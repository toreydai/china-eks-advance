# Lab03 — Kubecost 成本优化

#### 更新时间: 2026-07-16
#### 基于EKS版本: EKS 1.35

## 实验简介

演示在 EKS 上部署 Kubecost 实现 Pod / Namespace / Label 维度的成本可见性，对比 AWS 原生 Split Cost Allocation，配合 Karpenter Spot 实战降本。

> 本 Lab 与 Quickstart Demo09（Karpenter 节点伸缩）互补：Demo09 解决"少买节点"，本 Lab 解决"看清谁在花钱"。

> **中国区适配**：
> - Kubecost 镜像位于 `gcr.io/kubecost1/*`，需预先转存到 ECR 中国
> - Kubecost 2.9.x chart 面向 3.0 升级准备，会要求 finops agent / federated storage；本 Lab 使用最新可直接安装的 2.8.6
> - Spot Data Feed 在中国区不可用，Kubecost 使用静态 Spot 价格表
> - AWS Cost Explorer + Split Cost Allocation 在 cn-northwest-1 已可用
> - 复用 Lab01 的共享集群 `adv-shared`（Auto Mode），所以 Karpenter NodePool 用 `eks.amazonaws.com` NodeClass group，不是开源 Karpenter 的 `karpenter.k8s.aws`

**实验目标：**
- 掌握 Kubecost 在中国区私有 ECR 镜像下的部署方式
- 理解 Pod/Namespace/Label 维度成本归因，及与 AWS 原生 Split Cost Allocation 的交叉验证
- 能够独立完成从资源配置对比到 Spot 降本的成本优化闭环

**实验流程：**
1. 启用 AWS 原生 Split Cost Allocation
2. 部署 Kubecost
3. 访问 UI
4. 部署示例工作负载（不同资源配置）
5. 配合 Karpenter Spot 降本

**预计 AI 执行时长：** 15-20 分钟（不含等待 Kubecost 收集数据的 30 分钟-2 小时）

## 前提条件

- Lab01 已完成，共享集群 `adv-shared` 存活
- Kubecost 镜像（`kubecost/frontend`、`kubecost/cost-model`、`prom/prometheus`）已转存到 ECR 中国
- Kubecost Helm chart（`cost-analyzer-2.8.6.tgz`）已下载到 S3 工具桶
- 已在 Billing 控制台启用 Split Cost Allocation（见步骤 1，启用后 24 小时生效，建议提前做）

```bash
export AWS_PROFILE=cn
export AWS_REGION=cn-northwest-1
export AWS_DEFAULT_REGION=cn-northwest-1
export CLUSTER_NAME=adv-shared
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REGISTRY=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn
export KUBECOST_VERSION=2.8.6
export TOOLS_BUCKET=eks-tools-transfer-${ACCOUNT_ID}

aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}
```

---

## 步骤

### 1. 启用 AWS 原生 Split Cost Allocation

控制台路径：`Billing → Cost Explorer → Settings → Split Cost Allocation Data → 勾选 Amazon EKS`

**预期输出**：设置保存成功；启用后 24 小时内，Cost Explorer 可按 `eks:cluster-name` / `eks:namespace` / `eks:workload-name` 维度查看成本。

### 2. 部署 Kubecost

下载预置 Helm Chart：

```bash
aws s3 cp s3://${TOOLS_BUCKET}/cost-analyzer-${KUBECOST_VERSION}.tgz /tmp/ --region ${AWS_REGION}
```

**预期输出**：下载完成，`/tmp/cost-analyzer-2.8.6.tgz` 文件存在。

创建 values 文件（中国区适配）：

```bash
cat <<EOF > /tmp/kubecost-values.yaml
kubecostFrontend:
  image: ${ECR_REGISTRY}/kubecost/frontend
  imageVersion: prod-2.8.6

kubecostModel:
  image: ${ECR_REGISTRY}/kubecost/cost-model
  imageVersion: prod-2.8.6

prometheus:
  server:
    image:
      repository: ${ECR_REGISTRY}/prom/prometheus
      tag: v3.9.1
  alertmanager:
    enabled: false
  pushgateway:
    enabled: false
  nodeExporter:
    enabled: false
  kubeStateMetrics:
    enabled: false
  configmapReload:
    prometheus:
      enabled: false

grafana:
  enabled: false

kubecostProductConfigs:
  spotLabel: karpenter.sh/capacity-type
  spotLabelValue: spot
  awsSpotDataRegion: ${AWS_REGION}
  projectID: ${ACCOUNT_ID}

networkCosts:
  enabled: true

clusterController:
  enabled: false
EOF

kubectl create namespace kubecost

helm upgrade --install kubecost /tmp/cost-analyzer-${KUBECOST_VERSION}.tgz \
  --namespace kubecost \
  -f /tmp/kubecost-values.yaml \
  --wait \
  --timeout 10m

kubectl get pods -n kubecost
```

**预期输出**：`kubectl get pods -n kubecost` 中 `kubecost-cost-analyzer`（frontend + cost-model）与 Prometheus 相关 Pod 均 Running。

### 3. 访问 UI

```bash
kubectl port-forward -n kubecost svc/kubecost-cost-analyzer 9090:9090 &
```

**预期输出**：`Forwarding from 127.0.0.1:9090 -> 9090`。仅用 `kubectl port-forward` 从操作机本地访问，不创建 LoadBalancer/Ingress 暴露公网。

### 4. 部署示例工作负载（不同资源配置）

```bash
kubectl create namespace cost-demo
kubectl label namespace cost-demo team=engineering

# 应用 A：超额配置
kubectl create deployment over-provisioned \
  --image=public.ecr.aws/docker/library/nginx:1.27-alpine \
  --replicas=3 \
  -n cost-demo
kubectl set resources deployment/over-provisioned -n cost-demo \
  --requests=cpu=2,memory=4Gi --limits=cpu=2,memory=4Gi

# 应用 B：合理配置
kubectl create deployment right-sized \
  --image=public.ecr.aws/docker/library/nginx:1.27-alpine \
  --replicas=3 \
  -n cost-demo
kubectl set resources deployment/right-sized -n cost-demo \
  --requests=cpu=100m,memory=128Mi

# 应用 C：闲置资源
kubectl create deployment idle-app \
  --image=public.ecr.aws/docker/library/nginx:1.27-alpine \
  --replicas=2 \
  -n cost-demo
```

**预期输出**：`cost-demo` 命名空间下三个 Deployment（`over-provisioned`/`right-sized`/`idle-app`）全部 Running。Kubecost 至少需要 30 分钟收集数据后才能看到准确成本，完整建议（右 sizing / 闲置资源 / Spot 建议）需 1-2 小时。

### 5. 配合 Karpenter Spot 降本

`adv-shared` 是 Auto Mode 集群，Karpenter 内置，直接加自定义 Spot NodePool：

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spot
spec:
  template:
    metadata:
      labels:
        karpenter.sh/capacity-type: spot
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["t3.medium", "t3.large", "t3a.medium", "t3a.large", "m5.large", "m5a.large"]
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
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
EOF
```

**预期输出**：`nodepool.karpenter.sh/spot created`。`nodeClassRef` 用 Auto Mode 的 `eks.amazonaws.com` group（非开源 Karpenter 的 `karpenter.k8s.aws`）；`expireAfter` 用 `336h`（14 天），同 Lab01——加上默认 24h 优雅终止期不能超过 21 天。

把 `right-sized` 应用迁到 Spot：

```bash
kubectl patch deployment right-sized -n cost-demo --type=merge -p '
spec:
  template:
    spec:
      tolerations:
      - key: spot
        operator: Equal
        value: "true"
        effect: NoSchedule
      nodeSelector:
        karpenter.sh/capacity-type: spot
'

kubectl get nodes -L karpenter.sh/capacity-type
```

**预期输出**：约 1 分钟内出现一个 `CAPACITY-TYPE=spot` 的新节点，`right-sized` 的 Pod 重新调度到该节点。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 所有核心资源创建成功且状态正常
- [ ] 验证检查点中的所有命令返回预期结果
- [ ] 理解 Kubecost 成本归因维度与 AWS 原生 Split Cost Allocation 的对应关系

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get pods -n kubecost --no-headers \| grep Running \| wc -l \| tr -d ' '` | 大于等于 `2` |
| 2 | `kubectl get deployment -n cost-demo --no-headers \| wc -l \| tr -d ' '` | `3` |
| 3 | `kubectl get nodes -L karpenter.sh/capacity-type --no-headers \| grep -c spot` | 大于等于 `1` |
| 4 | `kubectl get pod -n cost-demo -l app=right-sized -o jsonpath='{.items[0].spec.nodeSelector.karpenter\.sh/capacity-type}'` | `spot` |

在 Kubecost UI（`http://localhost:9090`）查看：

- **Overview**：集群总成本、按 namespace 分摊
- **Allocations**：切到 namespace=cost-demo，对比三个 deployment 的成本
- **Savings**：自动识别右 sizing、闲置资源、Spot 建议

在 AWS 控制台 Cost Explorer 查看：

1. Group by `Resource` → `eks:namespace`
2. Filter `Service = Amazon EKS`
3. 看到 cost-demo namespace 的成本拆分

---

## 实验总结

本 Lab 验证了 Kubecost 在中国区（私有 ECR 镜像 + 无 Spot Data Feed）下的可用性，展示了从资源配置对比到 Spot 降本的完整成本优化闭环，并与 AWS 原生 Split Cost Allocation 做了交叉验证。Lab05 将学习 GenAI 推理服务。

---

## 清理

```bash
kubectl delete namespace cost-demo
kubectl delete nodepool spot

helm uninstall kubecost -n kubecost
kubectl delete namespace kubecost
```

**预期输出**：`cost-demo`/`kubecost` 命名空间和 `spot` NodePool 全部清除，Spot 节点自动回收。Cost Explorer / Split Cost Allocation 启用后无需清理（不产生额外费用）。共享集群 `adv-shared` 不删除，供后续 Lab 复用。
