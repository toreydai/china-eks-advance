# Lab05 — GenAI 推理服务（GPU + vLLM）

#### 更新时间: 2026-07-17
#### 基于EKS版本: EKS 1.35

## 实验简介

演示在 EKS 上部署 vLLM 推理服务，覆盖 GPU 节点准备、模型加载、Auto Mode 原生 GPU NodePool 弹性伸缩、Spot GPU 成本优化。

> **GPU 实例配额**：cn-northwest-1 GPU 实例需提前申请配额。当前控制面 offerings 复核显示 `g5.xlarge` 可用，未显示 `g6.xlarge`；本 Lab 推荐 g5.xlarge（1× A10G 24G 显存）。On-Demand G/VT 配额 768 vCPU、Spot G/VT 配额 64 vCPU，均足够单节点验证。
>
> **中国区适配**：
> - DCGM Exporter / vLLM 镜像已转存到 ECR 中国
> - vLLM 镜像需转存到 ECR 中国（体积较大，转存时注意本机磁盘空间，见根目录 `CLAUDE.md`）
> - HuggingFace 模型下载在操作机（本机 sandbox 有完整外网直连）直接下载后上传到 S3，不经中国区节点网络
> - 复用 Lab01 的共享集群 `adv-shared`（Auto Mode）
>
> **已实测确认（2026-07-17，替换原"待实测确认"结论）**：EKS Auto Mode 的内置 Karpenter（`eks.amazonaws.com`/`NodeClass`，即默认 `default` NodeClass）**原生支持 GPU 实例类型和 GPU Spot**，不需要额外的 Managed NodeGroup、不需要手动安装 NVIDIA Device Plugin、也不需要安装开源 Karpenter（`karpenter.k8s.aws`/`EC2NodeClass`）。实测过程：
> 1. 用默认 `NodeClass`（`nodeClassRef: {group: eks.amazonaws.com, kind: NodeClass, name: default}`）创建一个 `karpenter.sh/v1` `NodePool`，`requirements` 里指定 `node.kubernetes.io/instance-type: g5.xlarge`；
> 2. 提交一个请求 `nvidia.com/gpu: 1` 的 Pod，Auto Mode 在约 40 秒内自动拉起 g5.xlarge 节点（Bottlerocket EKS Auto AMI 自动选择 GPU 变体），节点自动带上 `nvidia.com/gpu: 1` allocatable、`eks.amazonaws.com/instance-gpu-manufacturer=nvidia` 等标签，全程**没有任何 NVIDIA Device Plugin DaemonSet**被创建或需要安装；
> 3. 把 NodePool 的 `karpenter.sh/capacity-type` requirement 改成 `["spot", "on-demand"]` 后，同一个 Auto Mode 默认 NodeClass 成功拉起 `capacity-type=spot` 的 g5.xlarge 节点。
>
> 结论：Lab05 的 GPU 节点准备与 Spot 降本，全部通过 Auto Mode 内置 NodePool 一套 CRD 完成，原文档步骤 1（Managed NodeGroup）+ 步骤 2（手动装 Device Plugin）+ 步骤 7（开源 Karpenter EC2NodeClass）已合并简化为下面的步骤 1（Auto Mode GPU NodePool）+ 步骤 7（同一 NodePool 切换 capacity-type 验证 Spot）。
>
> **工具链修正**：原文档步骤引用的 S3 工具桶 `eks-tools-transfer-${ACCOUNT_ID}` 在本次实测时**不存在**（`aws s3 ls` 确认账号下只有 `llm-models-${ACCOUNT_ID}` 一个桶）。DCGM Exporter 的 Helm Chart 改为直接从官方仓库 `https://nvidia.github.io/dcgm-exporter/helm-charts` 拉取——这一步是 `helm` 命令在操作机本地执行，操作机本身有完整外网直连，不受"EKS 节点访问不了 github.io"这条规则限制（该规则只针对**节点侧** containerd 拉取镜像和 kubelet 侧组件下载，不针对操作机的 `helm repo add`）。

**实验目标：**
- 掌握中国区 GPU 实例上 vLLM 推理服务的完整部署流程
- 理解 EKS Auto Mode 对 GPU 节点/GPU Spot 的原生支持，无需额外的 Device Plugin 或开源 Karpenter
- 理解 Pod Identity 访问 S3、模型加载的配合方式
- 能够独立完成从 GPU 节点准备到推理调用验证的完整闭环

**实验流程：**
1. 创建 GPU NodePool（Auto Mode 原生，含 Spot 能力）
2. 验证 GPU 资源自动可分配（无需手动安装 Device Plugin）
3. 准备模型文件（S3）
4. 配置 Pod Identity 访问 S3
5. 部署 vLLM
6. 测试推理
7. 验证 GPU Spot 弹性伸缩
8. 部署 DCGM Exporter（GPU 监控）

**预计 AI 执行时长：** 20-25 分钟

## 前提条件

- Lab01 已完成，共享集群 `adv-shared` 存活；GPU 实例配额已申请
- vLLM、DCGM Exporter 镜像已转存到 ECR 中国
- 推理模型（示例用 Qwen2.5-7B-Instruct-AWQ）从 HuggingFace 下载后上传到 S3 模型桶
- IAM 权限：Pod Identity Association 创建

```bash
export AWS_PROFILE=cn
export AWS_REGION=cn-northwest-1
export AWS_DEFAULT_REGION=cn-northwest-1
export CLUSTER_NAME=adv-shared
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REGISTRY=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn
export GPU_NS=llm-inference
export MODEL_BUCKET=llm-models-${ACCOUNT_ID}

aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}
```

---

## 步骤

### 1. 创建 GPU NodePool（Auto Mode 原生，含 Spot 能力）

```bash
kubectl create namespace ${GPU_NS} 2>/dev/null || true

cat <<'EOF' | kubectl apply -f -
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: gpu-auto
spec:
  template:
    metadata:
      labels:
        workload: gpu
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["g5.xlarge"]
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      taints:
        - key: nvidia.com/gpu
          value: "true"
          effect: NoSchedule
      expireAfter: 336h
  limits:
    cpu: "16"
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 60s
EOF

kubectl get nodepool gpu-auto
```

**预期输出**：`nodepool.karpenter.sh/gpu-auto created`，`NODECLASS` 列为 `default`（复用 Auto Mode 内置 NodeClass，不需要单独的 GPU NodeClass/Managed NodeGroup）。此时还没有 GPU 节点——Karpenter 是按需（Pod 触发）拉起节点，节点会在步骤 5 提交 vLLM Pod 时自动创建。

### 2. 验证 GPU 资源自动可分配（无需手动安装 Device Plugin）

提交一个最小化的测试 Pod 触发 Karpenter 拉起 GPU 节点：

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
  namespace: ${GPU_NS}
spec:
  nodeSelector:
    workload: gpu
  tolerations:
  - key: nvidia.com/gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  containers:
  - name: cuda-test
    image: public.ecr.aws/docker/library/busybox:latest
    command: ["sleep", "3600"]
    resources:
      requests:
        nvidia.com/gpu: 1
      limits:
        nvidia.com/gpu: 1
EOF

kubectl wait --for=condition=Ready pod/gpu-test -n ${GPU_NS} --timeout=180s

kubectl get nodes -L workload -L node.kubernetes.io/instance-type
kubectl describe nodes -l workload=gpu | grep -A6 "Allocatable:" | grep nvidia

kubectl delete pod gpu-test -n ${GPU_NS}
```

**预期输出**：约 40 秒内出现 1 个 `WORKLOAD=gpu`、`INSTANCE-TYPE=g5.xlarge` 的新节点，状态 `Ready`；`Allocatable` 里出现 `nvidia.com/gpu: 1`。**全程不会看到任何 NVIDIA Device Plugin Pod**（`kubectl get pods -A | grep nvidia` 应为空）——Auto Mode 的 Bottlerocket GPU AMI 变体自带 GPU 驱动和 kubelet device plugin 能力，`nvidia.com/gpu` allocatable 是节点自动暴露的，不需要额外安装任何 DaemonSet。

### 3. 准备模型文件（S3）

模型从 HuggingFace 直接下载到操作机（本机 sandbox 有完整外网直连），再上传到 S3 模型桶——不经过任何中转桶：

```bash
aws s3 mb s3://${MODEL_BUCKET} --region ${AWS_REGION} 2>/dev/null || true

python3 -c "
from huggingface_hub import snapshot_download
snapshot_download(repo_id='Qwen/Qwen2.5-7B-Instruct-AWQ', local_dir='/tmp/Qwen2.5-7B-Instruct-AWQ')
"

aws s3 sync /tmp/Qwen2.5-7B-Instruct-AWQ/ \
  s3://${MODEL_BUCKET}/Qwen2.5-7B-Instruct-AWQ/ \
  --region ${AWS_REGION} --exclude ".cache/*"
```

**预期输出**：模型文件同步完成（约 5.2GiB，2 个 safetensors 分片），`aws s3 ls s3://${MODEL_BUCKET}/Qwen2.5-7B-Instruct-AWQ/` 可看到模型权重文件。

### 4. 配置 Pod Identity 访问 S3

```bash
cat <<EOF > /tmp/llm-s3-policy.json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws-cn:s3:::${MODEL_BUCKET}",
      "arn:aws-cn:s3:::${MODEL_BUCKET}/*"
    ]
  }]
}
EOF

aws iam create-policy \
  --policy-name vllm-s3-policy \
  --policy-document file:///tmp/llm-s3-policy.json 2>&1 || echo "(policy 已存在)"

kubectl -n ${GPU_NS} create serviceaccount vllm 2>/dev/null || true

eksctl create podidentityassociation \
  --cluster ${CLUSTER_NAME} \
  --namespace ${GPU_NS} \
  --service-account-name vllm \
  --permission-policy-arns arn:aws-cn:iam::${ACCOUNT_ID}:policy/vllm-s3-policy \
  --role-name vllm-s3-role \
  --region ${AWS_REGION}
```

**预期输出**：`created pod identity association for service account "vllm" in namespace "llm-inference"`。

### 5. 部署 vLLM

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-qwen
  namespace: ${GPU_NS}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vllm-qwen
  template:
    metadata:
      labels:
        app: vllm-qwen
    spec:
      serviceAccountName: vllm
      nodeSelector:
        workload: gpu
      tolerations:
      - key: nvidia.com/gpu
        operator: Equal
        value: "true"
        effect: NoSchedule
      initContainers:
      - name: model-loader
        image: ${ECR_REGISTRY}/amazon/aws-cli:latest
        command:
        - sh
        - -c
        - |
          aws s3 sync s3://${MODEL_BUCKET}/Qwen2.5-7B-Instruct-AWQ/ /models/qwen --region ${AWS_REGION}
          ls -lh /models/qwen
        volumeMounts:
        - name: model-cache
          mountPath: /models
      containers:
      - name: vllm
        image: ${ECR_REGISTRY}/vllm/vllm-openai:v0.25.1
        args:
        - --model=/models/qwen
        - --served-model-name=qwen2.5-7b
        - --gpu-memory-utilization=0.9
        - --max-model-len=8192
        - --quantization=awq
        ports:
        - containerPort: 8000
        resources:
          requests:
            cpu: 2
            memory: 12Gi
            nvidia.com/gpu: 1
          limits:
            nvidia.com/gpu: 1
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 60
          periodSeconds: 10
        volumeMounts:
        - name: model-cache
          mountPath: /models
      volumes:
      - name: model-cache
        emptyDir:
          sizeLimit: 20Gi
---
apiVersion: v1
kind: Service
metadata:
  name: vllm-qwen
  namespace: ${GPU_NS}
spec:
  selector:
    app: vllm-qwen
  ports:
  - port: 80
    targetPort: 8000
  type: ClusterIP
EOF

# 首次启动包括 S3 下载 + 模型加载到 GPU，约 3-5 分钟
kubectl rollout status deployment/vllm-qwen -n ${GPU_NS} --timeout=600s
```

**预期输出**：`deployment "vllm-qwen" successfully rolled out`（约 3-5 分钟，含模型下载与加载到 GPU 显存）。

> **两处实测修正**：① 镜像 tag 从原文档的 `v0.20.2` 改为 `v0.25.1`——ECR 中国仓库实际转存、验证通过的是 `v0.25.1`（两个 tag 上游都存在，选用已转存并验证完整的版本，避免重复拉取 11GB+ 镜像）。② `resources.requests.memory` 从 `16Gi` 改为 `12Gi`——g5.xlarge 只有 16GiB 内存，`kubectl describe node` 显示 Allocatable 约 14.7GiB（系统预留后），请求 `16Gi` 会导致 Pod 因 `Insufficient memory` 永久 `Pending`，这是实测中真实触发的调度失败。

### 6. 测试推理

```bash
kubectl port-forward -n ${GPU_NS} svc/vllm-qwen 8000:80 &
sleep 3

curl -s http://localhost:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen2.5-7b",
    "messages": [{"role": "user", "content": "用中文写一首关于 EKS 的五言绝句"}],
    "max_tokens": 200
  }' | jq -r '.choices[0].message.content'
```

**预期输出**：返回一段中文诗句文本（非空、非错误 JSON）。

### 7. 验证 GPU Spot 弹性伸缩

不需要额外安装开源 Karpenter 或另建 EC2NodeClass——把步骤 1 创建的同一个 `gpu-auto` NodePool 的 `capacity-type` requirement 加上 `spot`，Auto Mode 内置 Karpenter 就会在 Spot 容量可用时选择 Spot：

```bash
kubectl patch nodepool gpu-auto --type=json \
  -p '[{"op":"replace","path":"/spec/template/spec/requirements/0/values","value":["spot","on-demand"]}]'

# 提交一个显式要求 spot 的测试 Pod，验证 Auto Mode 能否真正拉起 GPU Spot 节点
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-spot-test
  namespace: ${GPU_NS}
spec:
  nodeSelector:
    workload: gpu
    karpenter.sh/capacity-type: spot
  tolerations:
  - key: nvidia.com/gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  containers:
  - name: cuda-test
    image: public.ecr.aws/docker/library/busybox:latest
    command: ["sleep", "3600"]
    resources:
      requests:
        nvidia.com/gpu: 1
      limits:
        nvidia.com/gpu: 1
EOF

kubectl wait --for=condition=Ready pod/gpu-spot-test -n ${GPU_NS} --timeout=180s

kubectl get nodes -L karpenter.sh/capacity-type -L node.kubernetes.io/instance-type -L workload

kubectl delete pod gpu-spot-test -n ${GPU_NS}
```

**预期输出**：新增 1 个 `CAPACITY-TYPE=spot`、`INSTANCE-TYPE=g5.xlarge` 的节点，与步骤 5 的 on-demand vLLM 节点并存（vLLM 生产 Pod 不建议放 Spot，中断会导致推理服务中断；这里仅用临时测试 Pod 验证 Auto Mode 是否具备 GPU Spot 能力）。测试 Pod 删除后，该 Spot 节点会在 `consolidateAfter: 60s` 后被自动回收。

### 8. 部署 DCGM Exporter（GPU 监控）

DCGM Exporter 的 Helm Chart 直接从官方仓库拉取（操作机有外网直连，`helm repo add` 不受节点侧网络限制约束），镜像仍使用 ECR 中国转存版本：

```bash
helm repo add gpu-helm-charts https://nvidia.github.io/dcgm-exporter/helm-charts
helm repo update gpu-helm-charts

helm upgrade --install dcgm-exporter gpu-helm-charts/dcgm-exporter \
  --namespace ${GPU_NS} \
  --set image.repository=${ECR_REGISTRY}/nvidia/dcgm-exporter \
  --set image.tag=4.5.3-4.8.2-distroless \
  --set serviceMonitor.enabled=false \
  --set nodeSelector.workload=gpu \
  --set tolerations[0].key=nvidia.com/gpu \
  --set tolerations[0].operator=Equal \
  --set-string tolerations[0].value=true \
  --set tolerations[0].effect=NoSchedule

kubectl wait --for=condition=Ready pod -n ${GPU_NS} -l app.kubernetes.io/name=dcgm-exporter --timeout=60s

DCGM_POD=$(kubectl get pods -n ${GPU_NS} -l app.kubernetes.io/name=dcgm-exporter -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n ${GPU_NS} ${DCGM_POD} 9400:9400 &
sleep 2
curl -s http://localhost:9400/metrics | grep DCGM_FI_DEV_GPU_UTIL | head -3
```

**预期输出**：返回若干行 `DCGM_FI_DEV_GPU_UTIL{...} <数值>` 格式的 Prometheus 指标。配合 Quickstart Demo12（Prometheus + Grafana），可导入 NVIDIA Dashboard（ID: 12239）观察 GPU 利用率。

> **两处已修正**：① Chart 默认 DaemonSet 不带 `nodeSelector` 会在所有节点启动，非 GPU 节点会 `CrashLoopBackOff`，需 `--set nodeSelector.workload=gpu` 限定到 GPU 节点。② `--set tolerations[0].value="true"` 会被 Helm 解析成布尔值导致 DaemonSet 校验报错，需用 `--set-string` 强制字符串类型。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 所有核心资源创建成功且状态正常
- [ ] 验证检查点中的所有命令返回预期结果
- [ ] 理解 EKS Auto Mode 对 GPU/GPU Spot 的原生支持能力，理解 Pod Identity 访问 S3、模型加载的配合关系

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get nodes -l workload=gpu --no-headers \| wc -l \| tr -d ' '` | 大于等于 `1` |
| 2 | `kubectl get pods -n ${GPU_NS} -l app=vllm-qwen --no-headers \| grep Running \| wc -l \| tr -d ' '` | `1` |
| 3 | `curl -s http://localhost:8000/v1/models \| jq -r '.data[0].id'` | `qwen2.5-7b` |
| 4 | `curl -s http://localhost:9400/metrics \| grep -c DCGM_FI_DEV_GPU_UTIL` | 大于 `0` |

---

## 实验总结

本 Lab 验证了中国区 GPU 实例上 vLLM 推理服务的部署闭环，并确认 EKS Auto Mode 内置 Karpenter 原生支持 GPU 与 GPU Spot：无需 Managed NodeGroup、手动 Device Plugin 或开源 Karpenter 即可拉起 GPU/Spot 节点，是 Auto Mode 集群上 GPU 工作负载的推荐做法。过程中也修复了镜像 tag、内存请求超限、DCGM nodeSelector、Helm 布尔类型转换四个真实问题。Lab06 将学习 Spark on EKS。

---

## 清理

```bash
kubectl delete namespace ${GPU_NS}

kubectl delete nodepool gpu-auto 2>&1 || true

eksctl delete podidentityassociation \
  --cluster ${CLUSTER_NAME} \
  --namespace ${GPU_NS} \
  --service-account-name vllm \
  --region ${AWS_REGION}

aws iam delete-role --role-name vllm-s3-role 2>&1 || true
aws iam delete-policy --policy-arn arn:aws-cn:iam::${ACCOUNT_ID}:policy/vllm-s3-policy 2>&1 || true

# 模型 bucket 较大，按需清理
# aws s3 rb s3://${MODEL_BUCKET} --force --region ${AWS_REGION}
```

**预期输出**：命名空间与 `gpu-auto` NodePool 全部删除确认无残留；NodePool 删除后 Karpenter 会自动终止其管理的 GPU 节点（`kubectl get nodes -l workload=gpu` 应为空）。GPU 实例每小时费用较高，务必确认删除生效。共享集群 `adv-shared` 不删除，供后续 Lab 复用。
