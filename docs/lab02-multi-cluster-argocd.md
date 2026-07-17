# Lab02 — 多集群管理（ArgoCD Hub-Spoke）

#### 更新时间: 2026-07-16
#### 基于EKS版本: EKS 1.35

## 实验简介

演示用 ArgoCD Hub-Spoke 模式管理多个 EKS 集群：1 个 Hub 集群部署 ArgoCD，N 个 Spoke 集群作为部署目标。通过 ApplicationSet + Cluster Generator 实现"一份 Git 仓库，自动部署到所有集群"。

> **中国区适配**：
> - ArgoCD 镜像位于 `quay.io/argoproj/*`，需预先转存到 ECR 中国（本机 sandbox 可直接 `docker pull → tag → push`，无需搭建操作机中转，见根目录 `CLAUDE.md`）
> - Hub 集群复用本 workshop 的共享集群 `adv-shared`（Lab01 建成），Spoke 集群单独新建
> - 中国区 GitHub 访问不稳定（实测 `git ls-remote` 偶发 `context deadline exceeded`，重试 2-3 次通常能成功），如反复失败改用 CodeCommit 仓库，操作流程相同
> - **示例应用 `argocd-example-apps` 仓库里 `guestbook`/`helm-guestbook` 路径默认镜像是 `gcr.io/google-samples/gb-frontend:v5`，`gcr.io` 在中国区 EKS 节点侧不可达**，必须转存到 ECR 中国，并通过 Kustomize/Helm 的镜像覆盖机制指向 ECR 镜像（下方步骤 6/7/10 已内置此适配，不能直接用文档最初设想的裸 `guestbook` 路径）

**实验目标：**
- 掌握 ArgoCD Hub-Spoke 模式跨多个 EKS 集群的 GitOps 部署
- 理解 ApplicationSet + Cluster Generator 的按标签自动分发机制
- 能够独立完成从注册 Spoke 集群到验证多集群同步状态的完整闭环

**实验流程：**
1. 创建 Spoke 集群
2. 在 Hub 集群安装 ArgoCD
3. 安装 argocd CLI
4. 登录 ArgoCD
5. 注册 Spoke 集群到 ArgoCD
6. 部署示例应用到单个集群
7. ApplicationSet：自动部署到所有集群
8. 给 Hub 也打 prod 标签，验证自动部署
9. 验证两个集群都部署成功
10. 多集群差异化配置（Helm valueFiles）
11. 跨集群统一身份（IAM Access Entry）

**预计 AI 执行时长：** 25-30 分钟

## 前提条件

- Lab01 已完成，共享集群 `adv-shared` 存活（作为 Hub）
- `AWS_PROFILE=cn`，Helm 3.x，argocd CLI
- ArgoCD 镜像（`argoproj/argocd`、`library/redis`）已转存到 ECR 中国
- IAM 权限：EKS Access Entry、IAM Role 创建

```bash
export AWS_PROFILE=cn
export AWS_REGION=cn-northwest-1
export AWS_DEFAULT_REGION=cn-northwest-1
export HUB_CLUSTER=adv-shared
export SPOKE_CLUSTER=adv-spoke
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REGISTRY=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn
export ARGOCD_VERSION=3.4.5
export REDIS_TAG=7.4.9-bookworm
```

> **版本说明**：`ARGOCD_VERSION=3.4.5` 是实测时 `argo/argo-cd` Helm 仓库最新 chart（`10.1.4`）对应的 `appVersion`，`REDIS_TAG=7.4.9-bookworm` 是该 chart 默认依赖的 Redis 版本。执行前用 `helm search repo argo/argo-cd --versions | head -5` 确认当前最新版本号，二者保持一致即可（不必拘泥于某个具体版本号，但 ArgoCD Server 镜像 tag、Redis 镜像 tag、argocd CLI 版本三者要对齐，避免协议不兼容）。

---

## 步骤

### 1. 创建 Spoke 集群

```bash
eksctl create cluster \
  --name ${SPOKE_CLUSTER} \
  --region ${AWS_REGION} \
  --version 1.35 \
  --nodegroup-name ng-prod \
  --node-type t3.medium \
  --nodes 1 \
  --nodes-min 1 \
  --nodes-max 2 \
  --managed
```

**预期输出**：eksctl 输出集群创建成功，耗时约 15 分钟。此集群跑完本 Lab 后保留（供后续需要多集群场景的 Lab 复用），不在本 Lab 清理阶段删除。

### 2. 在 Hub 集群安装 ArgoCD

切换到 Hub 集群：

```bash
aws eks update-kubeconfig --name ${HUB_CLUSTER} --region ${AWS_REGION}
kubectl config current-context
```

**预期输出**：当前 context 切换为 Hub 集群 `adv-shared`。

镜像转存到 ECR 中国（先检查是否已转存，未转存再 pull/tag/push；本机 sandbox 有完整外网直连，不需要中转机）：

```bash
aws ecr describe-images --repository-name argoproj/argocd --region ${AWS_REGION} --query "imageDetails[?imageTags[0]=='v${ARGOCD_VERSION}']" 2>&1

# 如果上面为空（未转存），执行：
docker pull quay.io/argoproj/argocd:v${ARGOCD_VERSION}
docker tag quay.io/argoproj/argocd:v${ARGOCD_VERSION} ${ECR_REGISTRY}/argoproj/argocd:v${ARGOCD_VERSION}
docker push ${ECR_REGISTRY}/argoproj/argocd:v${ARGOCD_VERSION}
docker rmi quay.io/argoproj/argocd:v${ARGOCD_VERSION} ${ECR_REGISTRY}/argoproj/argocd:v${ARGOCD_VERSION}

docker pull redis:${REDIS_TAG}
docker tag redis:${REDIS_TAG} ${ECR_REGISTRY}/library/redis:${REDIS_TAG}
docker push ${ECR_REGISTRY}/library/redis:${REDIS_TAG}
docker rmi redis:${REDIS_TAG} ${ECR_REGISTRY}/library/redis:${REDIS_TAG}
```

**预期输出**：`aws ecr describe-images` 返回非空（已转存则跳过 pull/push）；否则两个镜像 push 成功，`docker images` 确认本地已 `rmi` 释放磁盘。

安装 ArgoCD（中国区镜像）：

```bash
kubectl create namespace argocd

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

cat <<EOF > /tmp/argocd-values.yaml
global:
  image:
    repository: ${ECR_REGISTRY}/argoproj/argocd
    tag: v${ARGOCD_VERSION}

dex:
  enabled: false

redis:
  image:
    repository: ${ECR_REGISTRY}/library/redis
    tag: ${REDIS_TAG}

server:
  service:
    type: ClusterIP
EOF

helm upgrade --install argocd argo/argo-cd \
  -n argocd \
  -f /tmp/argocd-values.yaml \
  --wait \
  --timeout 10m

kubectl rollout status deployment/argocd-server -n argocd --timeout=180s
kubectl get pods -n argocd
```

**预期输出**：`deployment "argocd-server" successfully rolled out`；`kubectl get pods -n argocd` 全部组件 Running。`server.service.type` 固定用 `ClusterIP`（中国区安全基线禁止 `0.0.0.0/0` 公网入站），后续通过 `kubectl port-forward` 访问 ArgoCD UI/API，不对外暴露 LoadBalancer。

### 3. 安装 argocd CLI

先检查 S3 工具桶是否已有预置的二进制：

```bash
aws s3 ls s3://eks-tools-transfer-${ACCOUNT_ID}/argocd-linux-amd64 --region ${AWS_REGION} 2>&1
```

**实测发现**：该文件在 S3 工具桶中不存在。本机 sandbox 有完整外网直连，直接从 GitHub Releases 下载与 Server 版本一致的二进制（CLI 版本需与 ArgoCD Server 版本对齐，避免 gRPC 协议不兼容）：

```bash
curl -sSL -o /tmp/argocd https://github.com/argoproj/argo-cd/releases/download/v${ARGOCD_VERSION}/argocd-linux-amd64
sudo install -m 555 /tmp/argocd /usr/local/bin/argocd
argocd version --client
```

**预期输出**：打印 argocd CLI 客户端版本号 `v${ARGOCD_VERSION}`。

### 4. 登录 ArgoCD

```bash
# 获取初始 admin 密码
ARGOCD_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)

# 端口转发（仅本地/操作机访问，不对外暴露）
kubectl port-forward -n argocd svc/argocd-server 8080:443 &
sleep 3

# CLI 登录
argocd login localhost:8080 \
  --username admin \
  --password "${ARGOCD_PWD}" \
  --insecure
```

**预期输出**：`'admin:login' logged in successfully`。

### 5. 注册 Spoke 集群到 ArgoCD

切到 Spoke 集群获取 context：

```bash
aws eks update-kubeconfig --name ${SPOKE_CLUSTER} --region ${AWS_REGION}
SPOKE_CONTEXT=$(kubectl config current-context)

# 切回 Hub
aws eks update-kubeconfig --name ${HUB_CLUSTER} --region ${AWS_REGION}
```

**预期输出**：`SPOKE_CONTEXT` 取到 Spoke 集群的 ARN 格式 context 名称，非空。

注册 Spoke 集群（ArgoCD 会自动在 Spoke 集群创建 ServiceAccount + RBAC）：

```bash
argocd cluster add ${SPOKE_CONTEXT} \
  --name ${SPOKE_CLUSTER} \
  --label env=prod \
  --label region=cn-northwest-1 \
  -y

argocd cluster list
```

**预期输出**：`argocd cluster list` 显示两行：`in-cluster`（Hub）和 `${SPOKE_CLUSTER}`。

### 6. 部署示例应用到单个集群

`argocd-example-apps` 仓库的 `guestbook` 路径是纯 YAML（Directory 类型），镜像硬编码为 `gcr.io/google-samples/gb-frontend:v5`，在中国区 EKS 节点不可达且无法用 CLI 覆盖镜像。改用同仓库的 `kustomize-guestbook` 路径（内置 `kustomization.yaml`），先转存镜像，再用 `--kustomize-image` 覆盖：

```bash
# 转存 guestbook 应用镜像（先检查是否已转存）
aws ecr describe-images --repository-name google-samples/gb-frontend --region ${AWS_REGION} 2>&1

# 如果不存在，创建仓库并转存：
aws ecr describe-repositories --repository-names google-samples/gb-frontend --region ${AWS_REGION} >/dev/null 2>&1 \
  || aws ecr create-repository --repository-name google-samples/gb-frontend --region ${AWS_REGION}
docker pull gcr.io/google-samples/gb-frontend:v5
docker tag gcr.io/google-samples/gb-frontend:v5 ${ECR_REGISTRY}/google-samples/gb-frontend:v5
docker push ${ECR_REGISTRY}/google-samples/gb-frontend:v5
docker rmi gcr.io/google-samples/gb-frontend:v5 ${ECR_REGISTRY}/google-samples/gb-frontend:v5

# 独立命名空间 demo-app（重要：不要用 default，见下方说明），避免和步骤 7/8 的 ApplicationSet 抢占资源所有权
argocd app create demo-app \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path kustomize-guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace demo-app \
  --sync-policy automated \
  --sync-option CreateNamespace=true

argocd app set demo-app --kustomize-image gcr.io/google-samples/gb-frontend=${ECR_REGISTRY}/google-samples/gb-frontend:v5

argocd app sync demo-app
argocd app get demo-app
```

**预期输出**：`argocd app get demo-app` 显示 `Health Status: Healthy`、`Sync Status: Synced`。

> **踩坑记录：** `demo-app` 若和 ApplicationSet 管理的 `--dest-namespace default` 共用命名空间，Hub 打上 `env=prod` 标签后会再生成一个 Application 争抢同一份 `Deployment/Service`，触发 `SharedResourceWarning` 并导致 `Sync Status` 反复横跳。修复：`demo-app` 固定用独立命名空间（如 `demo-app`），与 ApplicationSet 管理的命名空间物理隔离。
>
> 中国区 GitHub 访问不稳定：`argocd app create`/`app set`/`app sync` 报 `context deadline exceeded` 时重试 2-3 次即可，无需改用 CodeCommit。

### 7. ApplicationSet：自动部署到所有集群

ApplicationSet 用 Cluster Generator 遍历所有已注册的集群，自动为每个集群创建 Application。

```bash
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook-all-clusters
  namespace: argocd
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          env: prod
  template:
    metadata:
      name: 'guestbook-{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps.git
        targetRevision: HEAD
        path: kustomize-guestbook
        kustomize:
          images:
          - gcr.io/google-samples/gb-frontend=${ECR_REGISTRY}/google-samples/gb-frontend:v5
      destination:
        server: '{{server}}'
        namespace: default
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
EOF

kubectl get applicationset -n argocd
kubectl get application -n argocd
```

**预期输出**：`kubectl get application -n argocd` 自动出现一个名为 `guestbook-${SPOKE_CLUSTER}` 的 Application。

> 与步骤 6 同理，这里同样把 `path` 从 `guestbook` 改成 `kustomize-guestbook` 并加 `kustomize.images` 覆盖，原因见步骤 6 的踩坑记录（`gcr.io` 镜像不可达）。

### 8. 给 Hub 也打 prod 标签，验证自动部署

```bash
argocd cluster add $(kubectl config current-context) \
  --name in-cluster-hub \
  --label env=prod \
  --upsert -y

sleep 10
kubectl get application -n argocd
```

**预期输出**：多出一个 `guestbook-in-cluster-hub` Application。

### 9. 验证两个集群都部署成功

```bash
kubectl get pods -n default -l app=guestbook-ui
```

**预期输出**：Hub 上 guestbook 应用 Pod Running。

```bash
aws eks update-kubeconfig --name ${SPOKE_CLUSTER} --region ${AWS_REGION}
kubectl get pods -n default -l app=guestbook-ui
```

**预期输出**：Spoke 上 guestbook 应用同样部署成功，Pod Running。

```bash
aws eks update-kubeconfig --name ${HUB_CLUSTER} --region ${AWS_REGION}
```

**预期输出**：context 切回 Hub 集群。

### 10. 多集群差异化配置（Helm valueFiles）

实战中不同集群往往配置不同（副本数、镜像 tag、资源配额）。ApplicationSet 支持按集群名称选择 valueFile：

```bash
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: nginx-multi-env
  namespace: argocd
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          env: prod
  template:
    metadata:
      name: 'nginx-{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps.git
        targetRevision: HEAD
        path: helm-guestbook
        helm:
          valueFiles:
          - 'values.yaml'
          - 'values-{{name}}.yaml'  # 不同集群加载不同 values（本仓库暂无对应文件，ignoreMissingValueFiles 会跳过，不影响 Sync）
          ignoreMissingValueFiles: true
          parameters:
          - name: image.repository
            value: ${ECR_REGISTRY}/google-samples/gb-frontend  # helm-guestbook chart 默认镜像同样是 gcr.io，见步骤 6 说明，需覆盖
          - name: image.tag
            value: "v5"
      destination:
        server: '{{server}}'
        namespace: nginx-multi
      syncPolicy:
        automated:
          prune: true
        syncOptions:
        - CreateNamespace=true
EOF
```

**预期输出**：`applicationset.argoproj.io/nginx-multi-env created`，后续 `kubectl get application -n argocd` 会为每个匹配集群生成对应 Application，`Health Status` 应为 `Healthy`（`helm-guestbook` chart 的 `values.yaml` 默认镜像同样是 `gcr.io/google-samples/gb-frontend:v5`，用 `helm.parameters` 覆盖为 ECR 中国镜像，复用步骤 6 已转存的 `google-samples/gb-frontend:v5`）。

### 11. 跨集群统一身份（IAM Access Entry）

```bash
cat <<EOF > /tmp/multi-cluster-trust.json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws-cn:iam::${ACCOUNT_ID}:root"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name MultiClusterOps \
  --assume-role-policy-document file:///tmp/multi-cluster-trust.json 2>&1 || true

ROLE_ARN=arn:aws-cn:iam::${ACCOUNT_ID}:role/MultiClusterOps

# 新建 IAM Role 到可被 EKS create-access-entry 接受存在 15-30 秒的最终一致性延迟，
# 实测立即调用会报 "InvalidParameterException: The specified principalArn is invalid: invalid principal"
sleep 20

for CLUSTER in ${HUB_CLUSTER} ${SPOKE_CLUSTER}; do
  aws eks create-access-entry \
    --cluster-name ${CLUSTER} \
    --principal-arn ${ROLE_ARN} \
    --type STANDARD \
    --region ${AWS_REGION} 2>&1 || true

  aws eks associate-access-policy \
    --cluster-name ${CLUSTER} \
    --principal-arn ${ROLE_ARN} \
    --policy-arn arn:aws-cn:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
    --access-scope type=cluster \
    --region ${AWS_REGION}
done
```

**预期输出**：`aws eks list-access-entries` 对 Hub 和 Spoke 都能看到 `MultiClusterOps` 角色的 access entry。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 所有核心资源创建成功且状态正常
- [ ] 验证检查点中的所有命令返回预期结果
- [ ] 理解 ApplicationSet Cluster Generator 按标签自动分发部署的机制

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `argocd cluster list --output json \| jq 'length'` | `2`（in-cluster + Spoke，视是否已加 Hub prod 标签可能为 3） |
| 2 | `kubectl get application -n argocd --no-headers \| wc -l \| tr -d ' '` | 大于等于 `2` |
| 3 | `argocd app get demo-app -o json \| jq -r '.status.health.status'` | `Healthy` |
| 4 | `aws eks list-access-entries --cluster-name ${HUB_CLUSTER} --region ${AWS_REGION} --query 'length(accessEntries)' --output text` | 大于等于 `1` |

---

## 实验总结

本 Lab 验证了 ArgoCD Hub-Spoke 模式在中国区的多集群 GitOps 能力：ApplicationSet 按标签自动分发部署、跨集群差异化配置、以及基于 IAM Access Entry 的统一身份管理，全部在私有 ECR 镜像 + 无公网暴露的约束下完成。Lab03 将学习 Kubecost 成本优化。

---

## 清理

```bash
# 停止 port-forward（若还在前台/后台运行）
lsof -ti:8080 2>/dev/null | xargs -r kill

# ApplicationSet 默认会给生成的 Application 加 resources-finalizer，
# 删除 ApplicationSet 会级联删除 Application 及其部署的实际资源（实测约 15-20 秒完成，需等待后再确认）
kubectl delete applicationset guestbook-all-clusters -n argocd
kubectl delete applicationset nginx-multi-env -n argocd
sleep 20
kubectl get application -n argocd   # 确认只剩 demo-app

argocd app delete demo-app -y

# argocd cluster rm 必须带 -y，否则会等待交互式确认（stdin 非 tty 时直接报 EOF fatal 报错，不会真正删除）
argocd cluster rm ${SPOKE_CLUSTER} -y
argocd cluster rm in-cluster-hub -y 2>&1 || true

helm uninstall argocd -n argocd
kubectl delete namespace argocd

# argocd cluster rm 只删除 ArgoCD 自己保存的集群凭证，不会清理它在 Spoke 集群里创建的 argocd-manager
# ServiceAccount/ClusterRole/ClusterRoleBinding（ArgoCD 已知行为，非 bug），需要手动清理
aws eks update-kubeconfig --name ${SPOKE_CLUSTER} --region ${AWS_REGION}
kubectl delete clusterrolebinding argocd-manager-role-binding --ignore-not-found
kubectl delete clusterrole argocd-manager-role --ignore-not-found
kubectl delete sa argocd-manager -n kube-system --ignore-not-found

# demo-app/nginx-multi 是通过 CreateNamespace=true 自动创建的命名空间，
# Application/ApplicationSet 删除后命名空间本身不会自动清除，需要手动删（Hub + Spoke 都要检查）
kubectl delete namespace nginx-multi --ignore-not-found
aws eks update-kubeconfig --name ${HUB_CLUSTER} --region ${AWS_REGION}
kubectl delete namespace demo-app --ignore-not-found
kubectl delete namespace nginx-multi --ignore-not-found

for CLUSTER in ${HUB_CLUSTER} ${SPOKE_CLUSTER}; do
  aws eks delete-access-entry \
    --cluster-name ${CLUSTER} \
    --principal-arn ${ROLE_ARN} \
    --region ${AWS_REGION} 2>&1 || true
done
aws iam delete-role --role-name MultiClusterOps 2>&1 || true
```

**预期输出**：ArgoCD 命名空间和相关 ApplicationSet/Application 全部清除，Access Entry 和临时 IAM Role 删除确认无残留；Hub 上只剩 `default`/`kube-system` 等系统命名空间和 Lab03 的 `kubecost`/`cost-demo`（不受影响）；Spoke 集群保留但不再有 `argocd-manager` 相关 RBAC 残留。

> Hub 集群 `adv-shared` 由多个 Lab 复用，不在本 Lab 删除。Spoke 集群 `adv-spoke` 是否删除按当前"集群生命周期策略"执行——全部相关 Lab 跑完前保留：
> ```bash
> eksctl delete cluster --name ${SPOKE_CLUSTER} --region ${AWS_REGION}
> ```
