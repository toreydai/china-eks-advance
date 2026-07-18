# Lab14 — 把已有节点组从默认 ENI 切换到 VPC CNI Custom Networking

#### 更新时间: 2026-07-18
#### 基于EKS版本: EKS 1.35

## 实验简介

EKS 默认所有 Pod 与 Node 共享同一 VPC 子网的 IP 空间：每个节点的主 ENI 既用于节点自身通信，也从同一子网为 Pod 分配辅助 IP，大规模集群很容易把子网 IP 耗尽。本 Lab 模拟真实运维场景：**集群已经用默认 ENI 模式跑了一段时间，现在要在不重建集群的前提下切换到 Custom Networking**——先跑一个基线 Pod 确认当前 IP 来自主 CIDR，再开启 Custom Networking 配置，最后对已有节点组做滚动节点替换（cordon → drain → 终止实例，让 ASG 补一个新实例），验证新节点上的 Pod IP 已经切到 Pod 专用子网。

> **本 Lab 用独立集群 `eks-custom-net-demo`，不能在共享集群 `adv-shared` 上跑**：`adv-shared` 是 EKS Auto Mode，节点由 Karpenter 风格的 NodePool 动态供给，没有传统 Auto Scaling Group；本 Lab 第 7 步"终止 EC2 实例触发 ASG 补节点"这套滚动替换手法依赖经典托管节点组的 ASG，在 Auto Mode 下不成立。

**实验目标：**
- 理解为什么 Custom Networking 只对"新申请的辅助 ENI"生效，已运行节点不会自动切换
- 掌握在已有节点组上，通过滚动替换节点完成 Custom Networking 切换的操作流程
- 理解为什么 `ENIConfig` 必须覆盖节点组实际可能用到的**全部** AZ，而不是随便挑几个
- 能够用切换前后两次 Pod IP 的对比，验证切换真正生效

**实验流程：**
1. 创建独立测试集群（经典托管节点组，2 节点）
2. 部署切换前基线测试 Pod（确认 IP 来自主 CIDR）
3. 关联次要 CIDR 到 VPC
4. 为节点组覆盖的每个 AZ 创建 Pod 专用子网并绑定路由表
5. 为每个 AZ 创建 `ENIConfig` CRD
6. 在 vpc-cni addon 上开启 Custom Networking
7. 滚动替换现有节点组的节点
8. 部署切换后测试 Pod，对比 IP 网段变化
9. 删除独立集群

**预计 AI 执行时长：** 35-40 分钟（含集群创建、节点滚动替换等待）

## 前提条件

- **工具**：AWS CLI v2、eksctl、kubectl v1.35、jq
- **权限**：EC2（VPC/Subnet/RouteTable/AutoScaling）、EKS 权限（建议 AdministratorAccess）

```bash
export AWS_PROFILE=cn
export AWS_REGION=cn-northwest-1
export AWS_DEFAULT_REGION=cn-northwest-1
export AWS_PARTITION=aws-cn
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export CLUSTER_NAME=eks-custom-net-demo
export POD_CIDR=100.64.0.0/16
```

> `100.64.0.0/16`（CGNAT 保留段）是 AWS 官方文档给 Custom Networking 用的示例网段，不与常见 VPC 主 CIDR 冲突，仅用于演示；生产环境需按实际 IP 规划选择不冲突的网段。

---

## 步骤

### 1. 创建独立测试集群

新建一个跨全部可用 AZ 的经典托管节点组集群，2 个节点用于滚动替换演示。

```bash
eksctl create cluster \
  --name ${CLUSTER_NAME} \
  --region ${AWS_REGION} \
  --version 1.35 \
  --nodegroup-name ng-default \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 3 \
  --managed \
  --asg-access

aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}

export NODEGROUP=ng-default
export VPC_ID=$(aws eks describe-cluster --name ${CLUSTER_NAME} --region ${AWS_REGION} \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text)

# 关键：枚举节点组关联子网实际横跨的全部 AZ，而不是只挑 2 个
NG_SUBNETS=$(aws eks describe-nodegroup --cluster-name ${CLUSTER_NAME} --nodegroup-name ${NODEGROUP} \
  --region ${AWS_REGION} --query 'nodegroup.subnets' --output text)
AZS=($(aws ec2 describe-subnets --subnet-ids ${NG_SUBNETS} --region ${AWS_REGION} \
  --query 'Subnets[].AvailabilityZone' --output text | tr '\t' '\n' | sort -u))
echo "节点组覆盖的 AZ：${AZS[@]}"

kubectl get nodes -o wide
```

**预期输出**：集群创建成功（约 15 分钟），2 个节点 `Ready`；`AZS` 打印出节点组横跨的全部 AZ（通常是集群所在区域的 2-3 个 AZ）

### 2. 部署切换前基线测试 Pod

在做任何改动之前，先确认当前（默认 ENI 模式）Pod IP 落在集群主 CIDR，作为切换后的对比基线。

```bash
kubectl run pod-ip-before --image=public.ecr.aws/docker/library/nginx:1.27-alpine \
  --overrides="{\"spec\":{\"nodeSelector\":{\"eks.amazonaws.com/nodegroup\":\"${NODEGROUP}\"}}}"

kubectl wait --for=condition=Ready pod/pod-ip-before --timeout=2m
BEFORE_IP=$(kubectl get pod pod-ip-before -o jsonpath='{.status.podIP}')
echo "切换前 Pod IP：${BEFORE_IP}"
```

**预期输出**：`切换前 Pod IP：` 后跟随集群主 CIDR 段内的地址（不是 `100.64.` 开头）

> 该 Pod 会在第 7 步节点滚动替换时被 drain 掉（正常现象），提前记好 `BEFORE_IP` 的值就够了，不用保证它能活到最后。

### 3. 关联次要 CIDR 到 VPC

```bash
ASSOC_ID=$(aws ec2 associate-vpc-cidr-block \
  --vpc-id ${VPC_ID} \
  --cidr-block ${POD_CIDR} \
  --region ${AWS_REGION} \
  --query 'CidrBlockAssociation.AssociationId' --output text)

until [ "$(aws ec2 describe-vpcs --vpc-ids ${VPC_ID} --region ${AWS_REGION} \
  --query "Vpcs[0].CidrBlockAssociationSet[?AssociationId=='${ASSOC_ID}'].CidrBlockState.State" \
  --output text)" = "associated" ]; do
  echo "等待次要 CIDR 关联生效..."
  sleep 5
done
echo "次要 CIDR 已关联：${ASSOC_ID}"
```

**预期输出**：打印"次要 CIDR 已关联：vpc-cidr-assoc-xxxx"

### 4. 为节点组覆盖的每个 AZ 创建 Pod 专用子网并绑定路由表

复用现有节点子网所在的路由表（沿用其 NAT/IGW 出网路径），避免另起一套路由。按 `AZS` 数组循环创建，覆盖节点组实际横跨的全部 AZ，而不是只建 2 个。

```bash
NODE_SUBNET=$(aws eks describe-cluster --name ${CLUSTER_NAME} --region ${AWS_REGION} \
  --query 'cluster.resourcesVpcConfig.subnetIds[0]' --output text)
ROUTE_TABLE_ID=$(aws ec2 describe-route-tables --region ${AWS_REGION} \
  --filters "Name=association.subnet-id,Values=${NODE_SUBNET}" \
  --query 'RouteTables[0].RouteTableId' --output text)

POD_SUBNET_IDS=()
RTB_ASSOC_IDS=()
for i in "${!AZS[@]}"; do
  AZ=${AZS[$i]}
  CIDR="100.64.$((i * 32)).0/19"

  SUBNET_ID=$(aws ec2 create-subnet --vpc-id ${VPC_ID} --region ${AWS_REGION} \
    --cidr-block ${CIDR} --availability-zone ${AZ} \
    --query 'Subnet.SubnetId' --output text)
  aws ec2 create-tags --region ${AWS_REGION} --resources ${SUBNET_ID} \
    --tags Key=Name,Value=eks-pod-subnet-${AZ} Key=kubernetes.io/cluster/${CLUSTER_NAME},Value=shared \
           Key=Project,Value=eks-china-advance Key=Lab,Value=Lab14

  RTB_ASSOC=$(aws ec2 associate-route-table --subnet-id ${SUBNET_ID} --region ${AWS_REGION} \
    --route-table-id ${ROUTE_TABLE_ID} --query 'AssociationId' --output text)

  POD_SUBNET_IDS+=(${SUBNET_ID})
  RTB_ASSOC_IDS+=(${RTB_ASSOC})
  echo "AZ ${AZ} → Pod 子网 ${SUBNET_ID}（${CIDR}）"
done
```

**预期输出**：按 AZ 数量打印对应的 Pod 子网 ID（节点组横跨几个 AZ 就有几行）

### 5. 创建 ENIConfig CRD

`ENIConfig` 的 `metadata.name` 必须与 AZ 名称一致——这是后续 `ENI_CONFIG_LABEL_DEF=topology.kubernetes.io/zone` 让 VPC CNI 按节点所在 AZ 自动匹配对应子网的依据。**必须为 `AZS` 里的每一个 AZ 都创建，一个都不能少。**

```bash
NODE_SG=$(aws eks describe-cluster --name ${CLUSTER_NAME} --region ${AWS_REGION} \
  --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId' --output text)

for i in "${!AZS[@]}"; do
  cat <<EOF | kubectl apply -f -
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: ${AZS[$i]}
spec:
  securityGroups:
    - ${NODE_SG}
  subnet: ${POD_SUBNET_IDS[$i]}
EOF
done
```

**预期输出**：`eniconfig.crd.k8s.amazonaws.com/<az> created` × AZ 数量

> ⚠️ **漏建任何一个 AZ 的 ENIConfig，都会让落到那个 AZ 的新节点卡死**：症状是节点 `NotReady`、`kubectl describe node` 显示 `NetworkPluginNotReady: cni plugin not initialized`，`aws-node` Pod 反复 `CrashLoopBackOff`。因为 ASG 补节点时会按节点组配置的子网/AZ 分布调度，不保证落在旧节点所在的同一个 AZ。实测中，2 节点集群横跨 3 AZ，只建 2 个 AZ 的 ENIConfig 就精确复现了这个问题；补建第三个 AZ 的子网+ENIConfig 后，该节点的 `aws-node` 在下一次自动重试时即恢复 Ready，无需手动重启。

### 6. 在 vpc-cni addon 上开启 Custom Networking

```bash
aws eks update-addon \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name vpc-cni \
  --configuration-values '{"env":{"AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG":"true","ENI_CONFIG_LABEL_DEF":"topology.kubernetes.io/zone"}}' \
  --region ${AWS_REGION} \
  --resolve-conflicts OVERWRITE

until [ "$(aws eks describe-addon --cluster-name ${CLUSTER_NAME} \
  --addon-name vpc-cni --region ${AWS_REGION} \
  --query 'addon.status' --output text)" = "ACTIVE" ]; do
  echo "等待 vpc-cni addon ACTIVE..."
  sleep 10
done

kubectl rollout restart daemonset aws-node -n kube-system
kubectl rollout status daemonset aws-node -n kube-system
echo "Custom Networking 已开启"
```

**预期输出**：打印"Custom Networking 已开启"

> ⚠️ **只影响新分配的辅助 ENI**：配置生效后，`pod-ip-before` 所在节点不会自动切换——它的辅助 ENI 早已从旧子网分配好，重启 `aws-node` 只是让 CNI 插件加载新配置，不会重新分配已存在 Pod 的 IP。必须让节点重新加入集群、重新申请辅助 ENI，才会按 `ENIConfig` 从 Pod 专用子网取 IP，这正是下一步要做的事。

### 7. 滚动替换现有节点组的节点

逐个 cordon + drain 现有节点，再通过 Auto Scaling 终止对应 EC2 实例（不降低期望容量），ASG 会自动补一个新实例；新节点从加入集群起就按 Custom Networking 分配 Pod IP。

```bash
OLD_NODES=($(kubectl get nodes -l eks.amazonaws.com/nodegroup=${NODEGROUP} -o jsonpath='{.items[*].metadata.name}'))
echo "待替换节点：${OLD_NODES[@]}"

for NODE in "${OLD_NODES[@]}"; do
  kubectl cordon ${NODE}
  kubectl drain ${NODE} --ignore-daemonsets --delete-emptydir-data --force --timeout=120s

  INSTANCE_ID=$(kubectl get node ${NODE} -o jsonpath='{.spec.providerID}' | awk -F/ '{print $NF}')
  aws autoscaling terminate-instance-in-auto-scaling-group \
    --instance-id ${INSTANCE_ID} \
    --no-should-decrement-desired-capacity \
    --region ${AWS_REGION}
  echo "已终止 ${NODE} (${INSTANCE_ID})，ASG 将自动补齐替换节点"

  # 等待旧节点对象从集群中移除，再处理下一个，避免同时中断过多容量
  until ! kubectl get node ${NODE} &>/dev/null; do
    sleep 10
  done
done

# 等待新节点全部 Ready，数量恢复到替换前
DESIRED=${#OLD_NODES[@]}
until [ "$(kubectl get nodes -l eks.amazonaws.com/nodegroup=${NODEGROUP} --no-headers 2>/dev/null | grep -c ' Ready ')" -ge "${DESIRED}" ]; do
  echo "等待替换节点就绪..."
  sleep 15
done
echo "节点滚动替换完成"
```

**预期输出**：打印"节点滚动替换完成"，`kubectl get nodes` 中节点名称与替换前不同，且全部 `Ready`

> ⚠️ 此过程会短暂降低节点组可用容量（每次少 1 个节点）。生产环境替换更大规模节点组时，建议改用 `eksctl` 的 nodegroup 滚动升级机制或临时调大 `desired`/`max` 容量再逐台替换。

> ⚠️ **若替换后某个节点一直 NotReady**：先查 `kubectl describe node <node>` 的 Ready 条件和该节点上 `aws-node` Pod 的状态/日志。若看到 `cni plugin not initialized` 或 `aws-node` CrashLoopBackOff，大概率是这个节点所在 AZ 缺 `ENIConfig`（见第 5 步的警告）——回到第 4-5 步为这个 AZ 补建子网和 ENIConfig，`aws-node` 会在下次自动重试时自愈，不需要手动删 Pod 或重启节点。

### 8. 部署切换后测试 Pod，对比 IP 网段变化

```bash
kubectl run pod-ip-after --image=public.ecr.aws/docker/library/nginx:1.27-alpine \
  --overrides="{\"spec\":{\"nodeSelector\":{\"eks.amazonaws.com/nodegroup\":\"${NODEGROUP}\"}}}"

kubectl wait --for=condition=Ready pod/pod-ip-after --timeout=2m
AFTER_IP=$(kubectl get pod pod-ip-after -o jsonpath='{.status.podIP}')

echo "切换前 Pod IP（旧记录）：${BEFORE_IP}"
echo "切换后 Pod IP：${AFTER_IP}"
```

**预期输出**：`切换后 Pod IP` 以 `100.64.` 开头，与切换前记录的主 CIDR 地址明显不在同一网段

---

## 验收标准

完成本 Lab 后，你应当能够：
- [ ] 切换前基线 Pod 的 IP 确认落在集群主 CIDR
- [ ] VPC 已成功关联次要 CIDR `100.64.0.0/16`，`ENIConfig` CRD 覆盖节点组横跨的**全部** AZ
- [ ] vpc-cni addon 的 `AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG` 已设为 `true`
- [ ] 现有节点组的节点已完成滚动替换（节点名称与切换前不同），且全部 `Ready`
- [ ] 切换后 Pod 的 IP 落在 `100.64.0.0/16` 段，与切换前形成对比

---

## 验证检查点

| # | 检查命令 | 期望输出 |
|---|---------|---------|
| 1 | `aws ec2 describe-vpcs --vpc-ids $VPC_ID --region cn-northwest-1 --query "Vpcs[0].CidrBlockAssociationSet[?CidrBlock=='100.64.0.0/16'].CidrBlockState.State" --output text` | `associated` |
| 2 | `kubectl get eniconfig --no-headers \| wc -l \| tr -d ' '` | 等于 `${#AZS[@]}`（节点组覆盖的 AZ 数） |
| 3 | `kubectl get nodes --no-headers \| grep -v ' Ready '` | 空输出（无 NotReady 节点） |
| 4 | `kubectl get pod pod-ip-after -o jsonpath='{.status.podIP}' \| cut -d. -f1,2` | `100.64` |

---

## 实验总结

本 Lab 演示了在不重建集群的前提下，把已有节点组从默认 ENI 模式切换到 VPC CNI Custom Networking：先用基线 Pod 固定"切换前"的事实，再声明式地关联次要 CIDR、创建覆盖全部 AZ 的 `ENIConfig`、开启 addon 配置，最后通过 cordon/drain + 终止实例触发 ASG 补齐新节点，完成真正的切换。两个核心认知：① Custom Networking 不是"配置下发即生效"，而是绑定在节点生命周期上——只有重新申请辅助 ENI 的新节点才会按 `ENIConfig` 分配 Pod IP；② `ENIConfig` 必须覆盖节点组实际可能用到的**全部** AZ，漏一个就可能让 ASG 补的新节点直接卡死在 NotReady。这也是本 Lab 必须用经典托管节点组的独立集群、而不能在 Auto Mode 的 `adv-shared` 上做的原因——Auto Mode 的节点生命周期由 Karpenter 风格的 NodePool 管理，没有可供本 Lab 操作的传统 ASG。

---

## 清理

```bash
eksctl delete cluster --name ${CLUSTER_NAME} --region ${AWS_REGION}

for i in "${!POD_SUBNET_IDS[@]}"; do
  aws ec2 disassociate-route-table --association-id ${RTB_ASSOC_IDS[$i]} --region ${AWS_REGION} 2>/dev/null || true
  aws ec2 delete-subnet --subnet-id ${POD_SUBNET_IDS[$i]} --region ${AWS_REGION} 2>/dev/null || true
done

echo "清理完成：eks-custom-net-demo 集群及相关 Pod 子网已删除"
```

**预期输出**：`eks-custom-net-demo` 集群及关联 CloudFormation 栈删除确认无残留；Pod 专用子网清除确认无残留。VPC 本身随集群删除一并清理，无需单独 disassociate 次要 CIDR。
