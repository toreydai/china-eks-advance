# Lab01 — EKS Auto Mode 深度演示

#### 更新时间: 2026-07-16
#### 基于EKS版本: EKS 1.35（Auto Mode）

## 实验简介

EKS Auto Mode 在经典 EKS 之上把节点、网络、负载均衡、存储、Karpenter 等组件全部托管。本 Lab 演示从零创建 Auto Mode 集群、部署应用观察自动化能力，并对比 Quickstart Demo01 的经典模式。

> **中国区可用性（实测 2026-05-22 / 2026-07-16 复测）**：cn-northwest-1 已支持 Auto Mode（自 2025 年中起）。Auto Mode 节点 OS 为 Bottlerocket，自动滚动升级。

**实验目标：**
- 掌握 EKS Auto Mode 集群创建与核心自动化能力（节点/负载均衡/存储自动管理）
- 理解 Auto Mode 与经典 EKS 在组件托管方式上的差异
- 能够独立完成从建集群到验证自动扩缩容、动态存储的完整闭环

**实验流程：**
1. 创建 Auto Mode 集群
2. 经典 EKS vs Auto Mode 对比
3. 创建 auto-ebs-sc StorageClass
4. 部署应用：自动出 LB、自动出节点
5. 添加自定义 NodePool（Spot）
6. 动态 PV

**预计 AI 执行时长：** 15-20 分钟

## 前提条件

- AWS CLI 2.x，`AWS_PROFILE=cn`（中国区账号）
- eksctl ≥ 0.196.0（需支持 `autoModeConfig`）
- kubectl v1.35
- IAM 权限：EKS、EC2、IAM（含 OIDC provider）、ELB 相关操作权限
- 安全组要求：本 Lab 演示的 Service 一律使用 `internal` scheme，不对公网开放（中国区账号安全基线要求，禁止 `0.0.0.0/0` 入站）

```bash
export AWS_PROFILE=cn
export AWS_REGION=cn-northwest-1
export AWS_DEFAULT_REGION=cn-northwest-1
export CLUSTER_NAME=adv-shared
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

---

## 步骤

### 1. 创建 Auto Mode 集群

```bash
cat <<EOF > /tmp/auto-cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_REGION}
  version: "1.35"

iam:
  withOIDC: true

autoModeConfig:
  enabled: true
EOF

eksctl create cluster -f /tmp/auto-cluster.yaml
```

**预期输出**：eksctl 输出集群创建成功，耗时约 12-15 分钟。Auto Mode 自动配置 EBS CSI **driver**、VPC CNI、CoreDNS、kube-proxy、Karpenter、Load Balancer Controller、Pod Identity Agent 等核心组件。

```bash
aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}

kubectl get nodes
kubectl get pods -A
```

**预期输出**：`kubectl get nodes` 刚创建时无节点（Auto Mode 按需创建）；`kubectl get pods -A` 中 kube-system 命名空间的托管组件 Pod 均 Running。

### 2. 经典 EKS vs Auto Mode 对比

| 组件 | 经典 EKS（Demo01） | Auto Mode |
|------|---------------------|-----------|
| 节点 | ManagedNodeGroup | 托管，按需创建 |
| VPC CNI / EBS CSI / CoreDNS | addon | 内置 |
| Load Balancer Controller | Helm 安装 | 内置 |
| Karpenter | 手动安装 | 内置 |
| Pod Identity Agent | addon | 内置 |
| OS / AMI | AL2023 / Bottlerocket | Bottlerocket（托管升级） |

### 3. 创建 auto-ebs-sc StorageClass

Auto Mode 内置的是 EBS CSI **driver**（`ebs.csi.eks.amazonaws.com`，`kubectl get csidrivers` 可见），但不会自动创建 StorageClass，需要自己创建并指向这个 provisioner：

```bash
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: auto-ebs-sc
provisioner: ebs.csi.eks.amazonaws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
EOF

kubectl get sc
```

**预期输出**：`kubectl get sc` 列表中出现 `auto-ebs-sc`，`PROVISIONER` 列为 `ebs.csi.eks.amazonaws.com`。

### 4. 部署应用：自动出 LB、自动出节点

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-auto
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-auto
  template:
    metadata:
      labels:
        app: hello-auto
    spec:
      containers:
      - name: nginx
        image: public.ecr.aws/docker/library/nginx:1.27-alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
---
apiVersion: v1
kind: Service
metadata:
  name: hello-auto
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internal
spec:
  type: LoadBalancer
  selector:
    app: hello-auto
  ports:
  - port: 80
    targetPort: 80
EOF
```

**预期输出**：`deployment.apps/hello-auto created` 和 `service/hello-auto created`。Service scheme 固定用 `internal`（中国区安全基线禁止 `0.0.0.0/0` 公网入站）。观察和验证一律从集群内部发起，见下方"验证检查点"。

观察自动创建的资源：

```bash
# 节点自动出现（约 30-60 秒）
kubectl get nodes -w

# 内置 NodeClass / NodePool
kubectl get nodeclass
kubectl get nodepool

# 内部 NLB 自动创建
kubectl get svc hello-auto -w

LB_DNS=$(kubectl get svc hello-auto -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "LB_DNS=${LB_DNS}"
```

**预期输出**：约 30-60 秒后 `kubectl get nodes` 出现一个 `Ready` 的 on-demand 节点；约 1 分钟内 `kubectl get svc hello-auto` 的 `EXTERNAL-IP` 列出现内部 NLB 域名；`LB_DNS` 变量取到非空值。

### 5. 添加自定义 NodePool（Spot）

Auto Mode 默认 NodePool 仅 On-Demand。如需 Spot，创建自定义 NodePool：

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spot-pool
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      taints:
        - key: spot
          value: "true"
          effect: NoSchedule
      expireAfter: 336h
  limits:
    cpu: "100"
EOF

kubectl get nodepool
```

**预期输出**：`nodepool.karpenter.sh/spot-pool created`，`kubectl get nodepool` 列表中出现 `spot-pool`。

> **注意 1**：Auto Mode 下 `nodeClassRef` 使用 `eks.amazonaws.com` group（非开源 Karpenter 的 `karpenter.k8s.aws`）。
> **注意 2**：`expireAfter` 加上默认 `terminationGracePeriod`（24h）不能超过 21 天，所以这里用 `336h`（14 天）而不是 `720h`（30 天），否则 API 会拒绝创建。

### 6. 动态 PV

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: auto-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: auto-ebs-sc
  resources:
    requests:
      storage: 10Gi
EOF

kubectl get pvc auto-pvc -w
```

**预期输出**：`persistentvolumeclaim/auto-pvc created`；`auto-ebs-sc` 用 `WaitForFirstConsumer` 绑定模式，PVC 会先停在 `Pending`，需要一个挂载该 PVC 的 Pod 触发调度后才会变成 `Bound`（下一步用于验证连通性的 busybox Pod 即可触发）。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 所有核心资源创建成功且状态正常
- [ ] 验证检查点中的所有命令返回预期结果
- [ ] 理解 Auto Mode 与经典 EKS 在节点/负载均衡/存储自动化上的差异

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws eks describe-cluster --name ${CLUSTER_NAME} --region ${AWS_REGION} --query 'cluster.status' --output text` | `ACTIVE` |
| 2 | `kubectl get nodes --no-headers \| grep Ready \| wc -l \| tr -d ' '` | 大于等于 `1` |
| 3 | `kubectl get svc hello-auto -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' \| grep -c .` | `1`（非空） |
| 4 | `kubectl get nodepool spot-pool -o jsonpath='{.metadata.name}'` | `spot-pool` |
| 5 | `kubectl get pvc auto-pvc -o jsonpath='{.status.phase}'` | `Bound`（需先运行验证连通性的 Pod 触发绑定） |

集群内连通性验证（不用公网 curl，用集群内 busybox；不要用 `public.ecr.aws/docker/library/curlimages/curl`，ECR Public 的 Docker Official Images 命名空间下没有这个镜像）：

```bash
kubectl run verify-tmp --rm -it --image=public.ecr.aws/docker/library/busybox:latest --restart=Never -- \
  wget -qO- --timeout=5 http://${LB_DNS}
```

**预期输出**：返回 nginx 默认首页 HTML（含 `Welcome to nginx!`）。

---

## 实验总结

本 Lab 验证了 EKS Auto Mode 在中国区的可用性与核心自动化能力：节点自动创建/回收（Karpenter）、内部负载均衡自动创建、自定义 Spot NodePool、动态 EBS 存储绑定，全部在不暴露公网入口的前提下完成验证。Lab02 将学习多集群管理与 ArgoCD。

---

## 清理

```bash
kubectl delete deployment hello-auto
kubectl delete svc hello-auto
kubectl delete pvc auto-pvc
kubectl delete nodepool spot-pool
kubectl delete sc auto-ebs-sc

# 等待节点回收（约 30-60 秒）
kubectl get nodes
```

**预期输出**：约 1-2 分钟内 `kubectl get nodes` 回到无节点状态（Karpenter 自动回收）。Auto Mode 创建的 NLB / EBS 卷会随 Service / PVC 删除自动释放。

> **如果本集群还要被其他 Lab（Lab03/05/06/Optional-Istio）复用，不要执行 `eksctl delete cluster`**；仅删除本 Lab 自己创建的资源即可。全部相关 Lab 完成后再统一删除集群：
> ```bash
> eksctl delete cluster --name ${CLUSTER_NAME} --region ${AWS_REGION}
> ```
