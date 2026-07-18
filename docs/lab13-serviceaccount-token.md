# Lab13 — ServiceAccount 长期 Token 访问 EKS：一个安全权衡案例

#### 更新时间: 2026-07-18
#### 基于EKS版本: EKS 1.35

## 实验简介

真实客户场景：某客户的 Golang 自动化系统部署在阿里云 ECS 云主机上，需要通过 API 方式连接 AWS EKS 集群操作 K8s 资源。系统的 Client 端全网部署、出口 IP 不固定，绑定 IAM 身份的常规方案（IAM 用户 AKSK、Access Entry）都要求 Client 端能被 AWS 识别为某个 IAM 身份，而这个客户的 Client 端恰恰做不到这一点。最终采用的方案是"为 ServiceAccount 创建一个不依赖 IAM 的长期 Token"。

**本 Lab 不是在教一个推荐做法，而是拆解一个真实的权衡决策**：先复现这个方案本身，再对照 EKS 官方推荐的 Access Entry（QuickStart Demo12 已覆盖）逐条列出代价，最后给出"什么场景下才该用这个方案"的判断标准。

**实验目标：**
- 复现 ServiceAccount + 长期 Token 访问 EKS API 的完整流程
- 用"能否绑定 IAM 身份"这一个问题，判断该用 Access Entry 还是该用长期 Token
- 说清楚长期 Token 方案的安全代价，以及生产落地前必须补的收尾项

**实验流程：**
1. 设置环境变量
2. 创建 ServiceAccount 与 ClusterRoleBinding
3. 创建长期 Token 类型 Secret
4. 提取 Token 并写入独立 kubeconfig
5. 验证 Token 鉴权（且证明它不依赖 IAM）

**预计 AI 执行时长：** 10-15 分钟

## 前提条件

- **工具**：AWS CLI v2、kubectl v1.35
- **权限**：对目标 EKS 集群有管理权限（可创建 ClusterRoleBinding）
- **前提**：复用共享集群 `adv-shared`，无需独立集群（本 Lab 只操作 K8s API 对象，不涉及节点/节点组，Auto Mode 集群同样适用）

```bash
export AWS_PROFILE=cn
export AWS_REGION=cn-northwest-1
export AWS_DEFAULT_REGION=cn-northwest-1
export CLUSTER_NAME=adv-shared
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export AWS_PARTITION=aws-cn
export SA_NS=kube-system
export SA_NAME=lab13-token-demo
export SA_SECRET_NAME=lab13-token-demo-secret
export SA_KUBECONFIG=/tmp/lab13-sa-token-kubeconfig
```

> ⚠️ **本 Lab 会在共享集群上创建一个绑定 `cluster-admin` 的永不过期 Token**。`SA_NAME`/`SA_SECRET_NAME` 已加 `lab13-` 前缀避免和其它 Lab 的资源撞名，但风险不因为改名而消失——做完第 5 步验证后**立即执行清理**，不要把这个 Secret 留在共享集群里过夜。

---

## 步骤

### 1. 设置环境变量

已在"前提条件"完成，直接执行确认：

```bash
aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}
kubectl get svc
```

**预期输出**：正常列出 `kubernetes` service，说明当前 IAM 身份能正常访问集群（这是切换到 Token 之前的基线状态）

### 2. 创建 ServiceAccount 与 ClusterRoleBinding

原方案把 ServiceAccount 建在 `kube-system` 并绑定 `cluster-admin`，这是原始客户案例的真实配置——因为 Client 端要执行的具体 K8s 操作范围不固定，客户当时选择了最省事但权限最大的绑定方式。生产环境应改用专用 namespace + 按实际需要的操作范围收窄的自定义 ClusterRole。

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ${SA_NAME}
  namespace: ${SA_NS}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ${SA_NAME}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: ${SA_NAME}
    namespace: ${SA_NS}
EOF
```

**预期输出**：`serviceaccount/lab13-token-demo created`、`clusterrolebinding.rbac.authorization.k8s.io/lab13-token-demo created`

### 3. 创建长期 Token 类型 Secret

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: ${SA_SECRET_NAME}
  namespace: ${SA_NS}
  annotations:
    kubernetes.io/service-account.name: ${SA_NAME}
type: kubernetes.io/service-account-token
EOF

kubectl -n ${SA_NS} wait --for=jsonpath='{.data.token}' secret/${SA_SECRET_NAME} --timeout=30s
```

**预期输出**：`secret/lab13-token-demo-secret created`，随后 `secret/lab13-token-demo-secret condition met`

> 这个 Secret 的 `type: kubernetes.io/service-account-token` 是关键——K8s 会立即签发一个 Token 并写入 `.data.token`，而且**默认永不过期**（这和 EKS 节点自动挂载给 Pod 的那种短期投影 Token 完全是两回事，不要混淆）。

### 4. 提取 Token 并写入独立 kubeconfig

复制一份现有 kubeconfig 单独使用 Token 凭证，不覆盖默认 kubeconfig，避免污染当前会话的 IAM 身份鉴权方式。

```bash
TOKEN=$(kubectl -n ${SA_NS} get secret ${SA_SECRET_NAME} -o jsonpath='{.data.token}' | base64 --decode)

cp ~/.kube/config ${SA_KUBECONFIG}
KUBECONFIG=${SA_KUBECONFIG} kubectl config set-credentials ${SA_NAME} --token=${TOKEN}
KUBECONFIG=${SA_KUBECONFIG} kubectl config set-context --current --user=${SA_NAME}
```

**预期输出**：命令无报错；`${SA_KUBECONFIG}` 中当前 context 的 user 已切换为 `lab13-token-demo`

### 5. 验证 Token 鉴权，且证明它不依赖 IAM

关键验证点不是"kubectl 能不能跑"，而是**去掉 `AWS_PROFILE` 之后它还能不能跑**——如果去掉后依然成功，才能证明确实是 Token 在起作用，而不是 IAM exec 插件在兜底。

```bash
env -u AWS_PROFILE KUBECONFIG=${SA_KUBECONFIG} kubectl get svc
```

**预期输出**：正常列出 `kubernetes` service——在没有任何 AWS IAM 凭证的情况下，仅凭 Token 完成了 API Server 鉴权

---

## 权衡对照：这个方案 vs Access Entry

| | ServiceAccount 长期 Token（本 Lab） | Access Entry（QuickStart Demo12） |
|---|---|---|
| 前提条件 | 客户端不需要任何 AWS IAM 身份 | 客户端必须能被识别为某个 IAM 身份（User/Role） |
| 凭证生命周期 | **默认永不过期**，需手动轮换 | 依附 IAM 身份的临时凭证，天然有过期机制 |
| 权限收敛 | 手动创建 ClusterRole 才能收窄，容易图省事绑 `cluster-admin` | 可用 `associate-access-policy` 精确到 namespace 级只读/只写 |
| 审计能力 | CloudTrail 看不到 Token 的每次使用，只能看 K8s 审计日志 | CloudTrail 完整记录 `AssumeRole`/API 调用链路 |
| 撤销方式 | 删除 Secret 立即失效，但**必须记得去删** | 从 IAM 侧直接 disable/删除身份，撤销路径更符合企业日常流程 |
| 适用场景 | 客户端无法绑定 IAM 身份（如本 Lab 场景：出口 IP 不固定的第三方云主机） | 客户端能拿到 IAM 身份的所有场景，**官方推荐路径** |

**判断标准只有一条**：客户端能不能被 AWS 识别为某个 IAM 身份。能，就该用 Access Entry；不能，才轮到长期 Token 这个权宜方案——选它就意味着主动接受"用可撤销性换部署简单性"的代价，落地前必须做到：① 用最小权限 ClusterRole 替代 `cluster-admin`；② 在 EKS 集群 Endpoint Access 配置 CIDR 白名单，只放行客户端出口 IP 段；③ 建立 Token 定期轮换机制（删除并重建 Secret）。

---

## 验收标准

完成本 Lab 后，你应当能够：
- [ ] ServiceAccount `lab13-token-demo` 与 ClusterRoleBinding 已在集群中创建
- [ ] Secret `lab13-token-demo-secret` 中 `token` 字段非空
- [ ] 独立 kubeconfig 在**去掉 AWS_PROFILE** 后仍能用 Token 凭证正常执行 `kubectl get svc`
- [ ] 能说清楚这个方案和 Access Entry 之间的判断标准，而不是把它当成通用最佳实践

---

## 验证检查点

| # | 检查命令 | 期望输出 |
|---|---------|---------|
| 1 | `kubectl -n kube-system get sa lab13-token-demo -o jsonpath='{.metadata.name}'` | `lab13-token-demo` |
| 2 | `kubectl -n kube-system get secret lab13-token-demo-secret -o jsonpath='{.data.token}' \| base64 --decode \| wc -c` | 大于 `0` |
| 3 | `env -u AWS_PROFILE KUBECONFIG=/tmp/lab13-sa-token-kubeconfig kubectl get svc kubernetes -o jsonpath='{.metadata.name}'` | `kubernetes` |

---

## 实验总结

本 Lab 复现了「ServiceAccount 长期 Token」访问 EKS 的最小可行方案，并用去掉 `AWS_PROFILE` 的方式证明了 Token 鉴权确实独立于 IAM 生效。它的价值不在于"学会一个新技巧"，而在于建立一个判断框架：只有当客户端确实无法绑定 IAM 身份时，才该考虑这个方案，且要清楚知道自己在用可撤销性、审计能力换取部署简单性。多数场景下，QuickStart Demo12 的 Access Entry 才是应该优先推荐给客户的路径。

---

## 清理

```bash
kubectl -n ${SA_NS} delete secret ${SA_SECRET_NAME} --ignore-not-found
kubectl delete clusterrolebinding ${SA_NAME} --ignore-not-found
kubectl -n ${SA_NS} delete serviceaccount ${SA_NAME} --ignore-not-found
rm -f ${SA_KUBECONFIG}

echo "清理完成：共享集群 adv-shared 上不应再残留 lab13-token-demo 相关的 SA/Secret/ClusterRoleBinding"
```

**预期输出**：三条 `deleted` 确认；由于是共享集群，务必在做完验证后立即执行本清理，不要把带 `cluster-admin` 的长期 Token 留在 `adv-shared` 上。
