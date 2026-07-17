# Lab10 — EKS 集群升级与维护

#### 更新时间: 2026-07-16
#### 基于EKS版本: EKS 1.34 → 1.35（升级演示）

## 实验简介

演示 EKS 版本升级前检查、control plane 升级、managed node group 升级、addon 升级和基础维护动作。

本 Lab 新建一个 EKS 1.34 集群 `eks-upgrade-demo`，完整演示升级到 1.35 的全流程，演示结束后删除。共享集群/主集群全程不受影响。

> 升级是高风险操作。生产环境必须先在测试集群演练，并确认 AWS 中国区目标版本可用。EKS control plane 升级后不能降级。
>
> **中国区适配**：在操作机上通过 SSM Session Manager 运行命令时，用的是实例 IAM Role，`--profile cn` 不适用于 SSM 会话内部。

**实验目标：**
- 掌握 EKS 集群从 control plane 到 node group、addon 的完整升级流程
- 理解升级前检查（废弃 API 扫描）与升级中各阶段的验证方法
- 能够独立完成从建临时升级集群到回归验证的完整闭环

**实验流程：**
1. 创建 1.34 升级测试集群
2. 升级前检查
3. 升级 control plane（1.34 → 1.35）
4. 升级 managed node group
5. 升级 EKS add-ons
6. 维护动作：cordon / drain / uncordon
7. 回归验证

**预计 AI 执行时长：** 40-50 分钟

## 前提条件

- IAM 权限：EKS 集群/节点组/addon 全套操作权限
- 操作机（如通过 SSM 访问）确认当前使用的是实例角色而非本地 profile

```bash
export AWS_REGION=cn-northwest-1
export AWS_DEFAULT_REGION=cn-northwest-1
export UPGRADE_CLUSTER=eks-upgrade-demo
export TARGET_VERSION=1.35
export SOURCE_VERSION=1.34
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

---

## 步骤

### 1. 创建 1.34 升级测试集群

> 新建最小化集群：1 个节点，t3.medium，AL2023，用于演示升级路径。耗时约 15 分钟。

```bash
eksctl create cluster \
  --name ${UPGRADE_CLUSTER} \
  --region ${AWS_REGION} \
  --version ${SOURCE_VERSION} \
  --nodegroup-name ng-upgrade \
  --node-type t3.medium \
  --nodes 1 \
  --nodes-min 1 \
  --nodes-max 2 \
  --managed \
  --asg-access

aws eks update-kubeconfig \
  --region ${AWS_REGION} \
  --name ${UPGRADE_CLUSTER}

kubectl config current-context
kubectl get nodes -o wide
```

**预期输出**：集群创建成功（约 15 分钟），当前 context 切换为 `${UPGRADE_CLUSTER}`，1 个节点 `Ready`，版本 `v1.34.x`。

### 2. 升级前检查

```bash
aws eks describe-cluster \
  --region ${AWS_REGION} \
  --name ${UPGRADE_CLUSTER} \
  --query 'cluster.{version:version,status:status,platformVersion:platformVersion,authMode:accessConfig.authenticationMode}'

kubectl version
kubectl get nodes -o wide
kubectl get pods -A
kubectl get pdb -A

eksctl get nodegroup \
  --cluster ${UPGRADE_CLUSTER} \
  --region ${AWS_REGION}

aws eks list-addons \
  --region ${AWS_REGION} \
  --cluster-name ${UPGRADE_CLUSTER}
```

**预期输出**：`describe-cluster` 返回 `version: "1.34"`, `status: "ACTIVE"`；`list-addons` 列出当前已装 addon（如 `vpc-cni`/`kube-proxy`/`coredns`/`eks-pod-identity-agent`）。

检查废弃 API：

```bash
kubectl api-resources --verbs=list --namespaced -o name \
  | xargs -n 1 kubectl get --ignore-not-found -A
```

**预期输出**：命令执行完成不报错（本身不直接判断废弃 API，只是列出各类型资源存在情况）。如安装了 `kubent`、`pluto` 等工具，可用它们做更完整的 deprecated API 扫描。

### 3. 升级 control plane（1.34 → 1.35）

```bash
aws eks update-cluster-version \
  --region ${AWS_REGION} \
  --name ${UPGRADE_CLUSTER} \
  --kubernetes-version ${TARGET_VERSION}

# 等待 control plane 升级完成（约 10-15 分钟）
aws eks wait cluster-active \
  --region ${AWS_REGION} \
  --name ${UPGRADE_CLUSTER}

aws eks describe-cluster \
  --region ${AWS_REGION} \
  --name ${UPGRADE_CLUSTER} \
  --query 'cluster.{version:version,status:status}'
```

**预期输出**：约 7-15 分钟后返回 `{"version": "1.35", "status": "ACTIVE"}`。

### 4. 升级 managed node group

```bash
export NODEGROUP_NAME=$(eksctl get nodegroup \
  --cluster ${UPGRADE_CLUSTER} \
  --region ${AWS_REGION} \
  -o json | jq -r '.[0].Name')

aws eks update-nodegroup-version \
  --region ${AWS_REGION} \
  --cluster-name ${UPGRADE_CLUSTER} \
  --nodegroup-name ${NODEGROUP_NAME} \
  --kubernetes-version ${TARGET_VERSION}

aws eks wait nodegroup-active \
  --region ${AWS_REGION} \
  --cluster-name ${UPGRADE_CLUSTER} \
  --nodegroup-name ${NODEGROUP_NAME}

kubectl get nodes -o wide
```

**预期输出**：约 10-15 分钟后节点组升级完成，`kubectl get nodes` 显示新节点版本为 `v1.35.x`（旧节点被替换）。

### 5. 升级 EKS add-ons

```bash
for addon in vpc-cni kube-proxy coredns eks-pod-identity-agent; do
  if aws eks describe-addon \
    --region ${AWS_REGION} \
    --cluster-name ${UPGRADE_CLUSTER} \
    --addon-name ${addon} >/dev/null 2>&1; then
    aws eks update-addon \
      --region ${AWS_REGION} \
      --cluster-name ${UPGRADE_CLUSTER} \
      --addon-name ${addon} \
      --resolve-conflicts OVERWRITE || true
  fi
done

kubectl -n kube-system get pods
aws eks list-addons --region ${AWS_REGION} --cluster-name ${UPGRADE_CLUSTER}
```

**预期输出**：全部已安装 addon 触发升级并最终状态 `ACTIVE`；`kube-system` 下相关 Pod（`aws-node`/`kube-proxy`/`coredns`）Running。

### 6. 维护动作：cordon / drain / uncordon

```bash
export MAINT_NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')

kubectl cordon ${MAINT_NODE}
kubectl get nodes

kubectl drain ${MAINT_NODE} \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force

kubectl uncordon ${MAINT_NODE}
kubectl get nodes
```

**预期输出**：`cordon` 后节点状态变为 `Ready,SchedulingDisabled`；`uncordon` 后恢复 `Ready`。

如果 drain 被 PDB 阻塞，先检查：

```bash
kubectl get pdb -A
kubectl describe pdb -A
```

**预期输出**：单节点集群中 CoreDNS PDB（`minAvailable=1`）会阻止 coredns pod 被驱逐，因为 drain 后无处调度——这是**预期行为**，不代表操作失败。生产环境应使用多节点，或在维护窗口临时删除 PDB（需评估风险）。

### 7. 回归验证

```bash
kubectl get nodes
kubectl get pods -A

kubectl create namespace upgrade-smoke
kubectl -n upgrade-smoke create deployment nginx \
  --image=public.ecr.aws/docker/library/nginx:1.27-alpine
kubectl -n upgrade-smoke rollout status deployment/nginx
kubectl -n upgrade-smoke run curl --rm -it --restart=Never \
  --image=public.ecr.aws/docker/library/busybox:1.36 \
  -- wget -qO- http://nginx.upgrade-smoke.svc.cluster.local

kubectl delete namespace upgrade-smoke
```

**预期输出**：`deployment "nginx" successfully rolled out`；`wget` 返回 nginx 默认首页 HTML，证明升级后集群网络/调度功能正常。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 所有核心资源创建成功且状态正常
- [ ] 验证检查点中的所有命令返回预期结果
- [ ] 理解单节点集群下 CoreDNS PDB 阻塞 drain 属于预期行为而非故障

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws eks describe-cluster --region ${AWS_REGION} --name ${UPGRADE_CLUSTER} --query 'cluster.version' --output text` | `1.35` |
| 2 | `aws eks describe-nodegroup --region ${AWS_REGION} --cluster-name ${UPGRADE_CLUSTER} --nodegroup-name ${NODEGROUP_NAME} --query 'nodegroup.version' --output text` | `1.35` |
| 3 | `aws eks list-addons --region ${AWS_REGION} --cluster-name ${UPGRADE_CLUSTER} --query 'length(addons)' --output text` | 大于等于 `1` |
| 4 | `kubectl get nodes --no-headers \| grep -c Ready` | 大于等于 `1` |

---

## 实验总结

本 Lab 完整走完了 EKS 从 1.34 到 1.35 的升级流程（control plane → node group → addon → 维护动作 → 回归验证），并验证了单节点集群下 CoreDNS PDB 会阻塞 drain 这一常见误判点属于预期行为而非故障。Lab11 将学习 Kyverno 策略治理。

---

## 清理

演示完成后删除临时集群，恢复 kubeconfig 到主集群：

```bash
eksctl delete cluster \
  --name ${UPGRADE_CLUSTER} \
  --region ${AWS_REGION}

# 恢复到主集群（按实际主集群名称调整）
aws eks update-kubeconfig \
  --region ${AWS_REGION} \
  --name demo

kubectl config current-context
kubectl get nodes
```

**预期输出**：`eks-upgrade-demo` 集群及相关 CloudFormation 栈删除确认无残留，kubeconfig 恢复指向主集群。

> `eksctl delete cluster` 会清空 kubeconfig 的 current-context，删除后需要手动切换回主集群 context。
