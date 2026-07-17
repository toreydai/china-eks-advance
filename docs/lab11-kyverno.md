# Lab11 — Kyverno 策略治理

#### 更新时间: 2026-07-16
#### 基于EKS版本: EKS 1.35

## 实验简介

通过 Kyverno 演示 Kubernetes 策略治理，要求业务工作负载设置 CPU/Memory requests 和 limits。

> Pod Security Standards 基础演示保留在 quickstart Demo17。
>
> **中国区适配**：Kyverno Helm chart 默认 registry 是 `reg.kyverno.io`，仅用 `--set image.repository=...` 不足以覆盖——chart 会把 ECR path 拼在 `reg.kyverno.io/` 之后（如 `reg.kyverno.io/<ECR_REGISTRY>/kyverno/kyverno:v1.18.1`），导致所有 Pod `ImagePullBackOff`。必须用 `--set global.image.registry=<ECR>` 全局覆盖，并为各组件单独设置 `registry`（已在下方步骤直接写成正确用法）。

**实验目标：**
- 掌握 Kyverno 在中国区私有 ECR 镜像下的部署方式（多组件 registry 覆盖）
- 理解 ClusterPolicy 的 `validate` 规则如何拦截不合规 Pod
- 能够独立完成从安装 Kyverno 到验证策略拦截的完整闭环

**实验流程：**
1. 安装 Kyverno
2. 创建策略：要求 requests/limits
3. 验证策略拦截
4. 创建符合策略的 Pod

**预计 AI 执行时长：** 10-15 分钟

## 前提条件

- Kyverno chart（`kyverno-3.8.1.tgz`）与镜像（`kyverno/kyverno`、`kyverno/kyvernopre`、`kyverno/background-controller`、`kyverno/cleanup-controller`、`kyverno/reports-controller`，均为 v1.18.1）已转存到 ECR 中国

```bash
export AWS_PROFILE=cn
export AWS_REGION=cn-northwest-1
export AWS_DEFAULT_REGION=cn-northwest-1
export CLUSTER_NAME=demo
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REGISTRY=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn
export KYVERNO_NS=kyverno
export POLICY_NS=policy-demo
export TOOLS_BUCKET=eks-tools-transfer-${ACCOUNT_ID}
```

---

## 步骤

### 1. 安装 Kyverno

```bash
aws s3 cp s3://${TOOLS_BUCKET}/kyverno-3.8.1.tgz /tmp/ --region ${AWS_REGION}

helm upgrade --install kyverno /tmp/kyverno-3.8.1.tgz \
  --namespace ${KYVERNO_NS} --create-namespace \
  --set global.image.registry=${ECR_REGISTRY} \
  --set admissionController.initContainer.image.repository=kyverno/kyvernopre \
  --set admissionController.initContainer.image.tag=v1.18.1 \
  --set admissionController.container.image.repository=kyverno/kyverno \
  --set admissionController.container.image.tag=v1.18.1 \
  --set backgroundController.image.registry=${ECR_REGISTRY} \
  --set backgroundController.image.repository=kyverno/background-controller \
  --set backgroundController.image.tag=v1.18.1 \
  --set cleanupController.image.registry=${ECR_REGISTRY} \
  --set cleanupController.image.repository=kyverno/cleanup-controller \
  --set cleanupController.image.tag=v1.18.1 \
  --set reportsController.image.registry=${ECR_REGISTRY} \
  --set reportsController.image.repository=kyverno/reports-controller \
  --set reportsController.image.tag=v1.18.1 \
  --wait --timeout 5m

kubectl -n ${KYVERNO_NS} get pods
```

**预期输出**：`kubectl -n ${KYVERNO_NS} get pods` 显示 4 个组件（admission-controller/background-controller/cleanup-controller/reports-controller）Pod 均 `1/1 Running`。

### 2. 创建策略：要求 requests/limits

```bash
kubectl create namespace ${POLICY_NS}

cat <<'EOF' | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-requests-limits
spec:
  validationFailureAction: Enforce
  background: true
  rules:
  - name: validate-resources
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - policy-demo
    validate:
      message: "CPU and memory requests/limits are required."
      pattern:
        spec:
          containers:
          - name: "*"
            resources:
              requests:
                cpu: "?*"
                memory: "?*"
              limits:
                cpu: "?*"
                memory: "?*"
EOF

kubectl get clusterpolicy require-requests-limits
```

**预期输出**：`kubectl get clusterpolicy require-requests-limits` 的 `READY` 列为 `True`。

### 3. 验证策略拦截

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-resources
  namespace: ${POLICY_NS}
spec:
  containers:
  - name: app
    image: public.ecr.aws/docker/library/nginx:1.27-alpine
EOF
```

**预期输出**：Kyverno 拒绝创建，报错信息包含 `resource Pod/policy-demo/no-resources was blocked due to the following policies` 和策略 message `CPU and memory requests/limits are required.`。

### 4. 创建符合策略的 Pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: with-resources
  namespace: ${POLICY_NS}
spec:
  containers:
  - name: app
    image: public.ecr.aws/docker/library/nginx:1.27-alpine
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi
EOF

kubectl -n ${POLICY_NS} get pod with-resources
kubectl get policyreport -A || true
```

**预期输出**：`with-resources` Pod 创建成功并 `Running`。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 所有核心资源创建成功且状态正常
- [ ] 验证检查点中的所有命令返回预期结果
- [ ] 理解 Kyverno Helm chart 多组件 registry 覆盖为什么不能只用 `image.repository`

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl -n ${KYVERNO_NS} get pods --no-headers \| grep Running \| wc -l \| tr -d ' '` | `4` |
| 2 | `kubectl get clusterpolicy require-requests-limits -o jsonpath='{.status.ready}'` | `true` |
| 3 | `kubectl apply -f - <<< '{"apiVersion":"v1","kind":"Pod","metadata":{"name":"no-resources-check","namespace":"'${POLICY_NS}'"},"spec":{"containers":[{"name":"app","image":"public.ecr.aws/docker/library/nginx:1.27-alpine"}]}}' 2>&1 \| grep -c blocked` | 大于等于 `1` |
| 4 | `kubectl -n ${POLICY_NS} get pod with-resources -o jsonpath='{.status.phase}'` | `Running` |

---

## 实验总结

本 Lab 验证了 Kyverno 在中国区私有 ECR 镜像下的策略治理能力，重点记录了 Helm chart 多组件 registry 覆盖这个容易漏配的环节（`image.repository` 不够，必须配合 `global.image.registry`）。作为主线 Lab 的最后一个，完成本 Lab 后可选做 Lab12 Istio 服务网格。

---

## 清理

```bash
kubectl delete namespace ${POLICY_NS}
kubectl delete clusterpolicy require-requests-limits
helm uninstall kyverno -n ${KYVERNO_NS}
kubectl delete namespace ${KYVERNO_NS}
```

**预期输出**：`policy-demo`/`kyverno` 命名空间及 ClusterPolicy 全部清除确认无残留。
