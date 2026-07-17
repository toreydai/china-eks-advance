# 架构文档

本仓库包含 12 个 Lab（`docs/lab01-*.md` ~ `docs/lab12-*.md`），这里不做全量架构图汇总，只对其中组件交互较复杂、值得可视化的 Lab 提供架构图；其余 Lab 请直接看对应的 `docs/labNN-*.md`。

选中的 5 个 Lab：

- **Lab02** — 多集群管理（ArgoCD Hub-Spoke）：跨 2 个 EKS 集群的 GitOps 分发，涉及 Hub/Spoke 集群、ApplicationSet 控制器、跨集群 IAM 身份
- **Lab05** — GenAI GPU 推理（vLLM）：Auto Mode 内置 Karpenter 原生支持 GPU/GPU Spot（免 Device Plugin），配合 Pod Identity 从 S3 加载真实模型
- **Lab06** — Spark on EKS：Spark Operator（开源）与 EMR on EKS（托管）两条数据处理路径并行，外加 Karpenter Spot 混合调度
- **Lab08** — CodePipeline 进行 EKS CI/CD：CodeCommit → CodePipeline → CodeBuild → ECR → EKS 的跨 AWS 服务交付链路
- **Lab12** — Istio 服务网格：中国区私有镜像仓库部署 + 内部 NLB/ClusterIP 验证 + Kiali/Jaeger 可观测性的完整闭环

其余 Lab（Lab01 Auto Mode、Lab03 Kubecost、Lab07 Gateway API+LBC、Lab09 OpenSearch+Fluent Bit、Lab10 集群升级、Lab11 Kyverno）以单一组件/单一操作链路为主，不再单独画图。

---

## Lab02 — 多集群管理（ArgoCD Hub-Spoke）

Hub 集群 `adv-shared` 部署 ArgoCD Server；Spoke 集群 `adv-spoke` 作为部署目标注册进 ArgoCD。ApplicationSet 用 Cluster Generator 按 `env=prod` 标签自动为每个已注册集群生成 Application，实现"一份 Git 仓库，自动同步到所有打标集群"；同时给 Hub 集群补打 `env=prod` 标签后，ApplicationSet 会自动再为 Hub 生成一个 Application，验证自动分发机制。中国区所有镜像（ArgoCD、Redis、guestbook 应用镜像）都需先转存到私有 ECR，示例应用改用 `kustomize-guestbook` 路径以支持镜像覆盖。

```mermaid
flowchart TB
  Op["操作机: argocd CLI"]
  ECR["私有 ECR 中国\n(argocd/redis/gb-frontend 镜像)"]
  Git["GitHub argocd-example-apps\n(kustomize-guestbook 路径)"]
  IAM["IAM Role MultiClusterOps\n+ Access Entry (Hub & Spoke)"]

  subgraph Hub["Hub 集群 adv-shared"]
    ArgoServer["argocd-server + redis\n(namespace argocd, ClusterIP)"]
    AppSet["ApplicationSet Controller\nCluster Generator (selector env=prod)"]
    HubApp["Application guestbook-in-cluster-hub\n(打 prod 标签后自动生成)"]
  end

  subgraph Spoke["Spoke 集群 adv-spoke"]
    SpokeApp["Application guestbook-adv-spoke\n(自动生成)"]
  end

  Op -->|1 helm install + port-forward 登录| ArgoServer
  Op -->|2 argocd cluster add spoke| AppSet
  Op -->|3 kubectl apply ApplicationSet| AppSet
  AppSet -->|按标签匹配已注册集群| HubApp
  AppSet -->|按标签匹配已注册集群| SpokeApp
  HubApp -->|sync| Git
  SpokeApp -->|sync| Git
  ArgoServer -->|拉取控制面镜像| ECR
  HubApp -->|拉取应用镜像| ECR
  SpokeApp -->|拉取应用镜像| ECR
  IAM -.->|create-access-entry 统一运维身份| Hub
  IAM -.->|create-access-entry 统一运维身份| Spoke
```

集群注册与自动分发的时序（ApplicationSet 如何在打标后自动补建 Application）：

```mermaid
sequenceDiagram
  participant Op as 操作机(argocd CLI)
  participant Hub as Hub集群 ArgoCD Server
  participant AppSet as ApplicationSet Controller
  participant Spoke as Spoke集群

  Op->>Hub: argocd cluster add ${SPOKE_CONTEXT} --label env=prod
  Op->>Hub: kubectl apply ApplicationSet(Cluster Generator, selector env=prod)
  AppSet->>Hub: 查询已注册且匹配标签的集群列表
  AppSet->>Spoke: 生成 Application guestbook-adv-spoke
  Spoke->>Spoke: sync 部署 kustomize-guestbook（拉取 ECR 镜像）
  Op->>Hub: argocd cluster add in-cluster --label env=prod（给 Hub 补打标签）
  AppSet->>Hub: 检测到新匹配集群，生成 Application guestbook-in-cluster-hub
  Hub->>Hub: sync 部署 kustomize-guestbook
```

## Lab05 — GenAI GPU 推理（vLLM）

与 Managed NodeGroup + 手动 Device Plugin 的传统方式不同，本 Lab 确认 **EKS Auto Mode 内置 Karpenter 原生支持 GPU 与 GPU Spot**：只需一个引用 Auto Mode 默认 NodeClass 的 `NodePool`，Pod 提交后即可按需拉起 GPU 节点，全程不需要安装 NVIDIA Device Plugin；NodePool 加上 `spot` capacity-type 即可择机拉起 Spot GPU 节点。真实的 Qwen2.5-7B-AWQ 模型从操作机下载后上传 S3，vLLM Pod 通过 initContainer 用 Pod Identity 凭证从 S3 同步模型再启动推理。

```mermaid
flowchart TB
  NodePool["Karpenter NodePool gpu-auto\n(nodeClassRef=Auto Mode 默认 NodeClass\ncapacity-type: on-demand→追加spot)"]
  GPUNode["GPU 节点\n(g5.xlarge, 按需自动拉起,\nnvidia.com/gpu allocatable 自动暴露, 免 Device Plugin)"]
  HF["HuggingFace\nQwen2.5-7B-Instruct-AWQ"]
  Ops["操作机(完整外网直连)"]
  S3Model["S3: llm-models-<acct>\n(模型权重 ~5.2GiB)"]
  PodID["Pod Identity: vllm SA → vllm-s3-role\n(S3 GetObject/ListBucket)"]
  InitC["initContainer model-loader\n(aws s3 sync → /models/qwen)"]
  vLLM["容器 vllm\n(--model=/models/qwen --quantization=awq)"]
  DCGM["DCGM Exporter\n(nodeSelector workload=gpu)"]

  NodePool -->|Pod触发按需拉起| GPUNode
  HF -->|snapshot_download| Ops -->|s3 sync| S3Model
  PodID --> InitC
  S3Model -->|s3 sync| InitC --> vLLM
  GPUNode --> vLLM
  GPUNode --> DCGM
```

---

## Lab06 — Spark on EKS

对比开源自建（Spark Operator）与 AWS 托管（EMR on EKS）两条数据处理路径，并演示 Driver 常驻节点 + Executor Spot 的成本优化。Spark Operator 路径受限于官方 `spark-py` 镜像缺 `hadoop-aws`，改用 ConfigMap 挂载脚本 + 集群内合成数据（`local://`），不走 S3；EMR on EKS 路径走 IRSA 凭证，完整验证真实 S3 读写；第三条路径把 Executor 调度到 Auto Mode 内置 NodePool 创建的 Spot 节点上，Driver 仍在普通节点。

```mermaid
flowchart TB
  S3["S3 Bucket spark-data\n(input/scripts/output-emr/output-spot)"]

  subgraph Cluster["EKS Auto Mode 集群 adv-shared, namespace spark-jobs"]
    Operator["spark-operator-controller\n(namespace spark-operator)"]
    SparkSynthetic["SparkApplication nyc-taxi-analysis\n(ConfigMap挂载脚本, local://, 集群内合成数据)"]
    SparkSpot["SparkApplication nyc-taxi-spot\n(mainApplicationFile s3a://, IRSA凭证)"]
    NodePool["Karpenter NodePool spark-spot\n(Auto Mode 内置 NodeClass, taint spark-spot)"]
    PodIdentity["Pod Identity: spark SA → spark-s3-role"]
  end

  subgraph EMR["EMR on EKS (托管)"]
    VC["EMR Virtual Cluster spark-on-eks-vc\n(container provider = Cluster/spark-jobs)"]
    EMRJob["EMR Job Run\n(entryPoint = S3 脚本, release emr-7.5.0)"]
    EMRRole["EMR Job Execution Role\n(IRSA WebIdentityTokenCredentialsProvider, S3 RW)"]
  end

  Operator --> SparkSynthetic
  PodIdentity -.->|S3 读写权限（本路径未使用，仅Spot路径用）| SparkSpot
  SparkSpot --> NodePool
  NodePool -->|capacity-type spot, g5系列外的通用实例| SparkSpot
  SparkSpot -->|s3a:// 读写| S3
  Cluster -.->|EKS container provider| VC
  VC --> EMRJob
  EMRRole -.->|update-role-trust-policy 绑定 VC namespace| EMRJob
  EMRJob -->|s3://读写, overwrite模式| S3
```

## Lab08 — CodePipeline 进行 EKS CI/CD

基于中国区原生服务（CodeCommit + CodeBuild + CodePipeline + ECR）构建 CI/CD 闭环：开发者推送代码触发 CodePipeline，CodeBuild 用 `buildspec.yml` 构建镜像并推送 ECR，再用绑定了 EKS Access Entry 的独立 IAM Role 对集群执行 `kubectl apply` 完成部署。CodeBuild 自身的服务角色（拉代码/推镜像/写日志）与它用来操作 EKS 的角色是两个独立 Role，职责分离。

```mermaid
flowchart LR
  Dev["开发者操作机\n(git-remote-codecommit 协议 push)"]
  CC["CodeCommit 仓库\neks-workshop-sample-api-service-go"]
  CP["CodePipeline demo-codepipeline\n(Source → Build)"]
  CB["CodeBuild Project demo-codebuild\n(buildspec.yml: docker build/push + kubectl apply)"]
  ECR["ECR demo-codepipeline"]
  EKS["EKS 集群 demo"]
  KubeRole["IAM Role EksWorkshopCodeBuildKubectlRole\n(+ EKS Access Entry AmazonEKSClusterAdminPolicy)"]
  BuildRole["IAM Role EksWorkshopCodeBuild\n(ecr/s3/codecommit/logs)"]
  PipeRole["IAM Role EksWorkshopCodePipeline"]

  Dev -->|git push main| CC
  CC -->|PollForSourceChanges 触发| CP
  CP -->|assume| PipeRole
  CP -->|Build Action| CB
  CB -->|assume| BuildRole
  CB -->|docker build/push| ECR
  CB -->|kubectl apply, 生成 imagedefinitions.json| EKS
  CB -.->|sts:AssumeRole 操作集群时使用| KubeRole
  KubeRole -.->|Access Entry 授权| EKS
```

推送代码到自动部署完成的时序：

```mermaid
sequenceDiagram
  participant Dev as 开发者
  participant CC as CodeCommit
  participant CP as CodePipeline
  participant CB as CodeBuild
  participant ECR as ECR
  participant EKS as EKS集群 demo

  Dev->>CC: git push origin main (修改 main.go)
  CC->>CP: PollForSourceChanges 触发新执行
  CP->>CB: 启动 Build Action（拉取 SourceArtifact）
  CB->>CB: docker build 镜像
  CB->>ECR: docker push 镜像
  CB->>EKS: kubectl apply（使用 EksWorkshopCodeBuildKubectlRole 授权）
  CB->>CP: post_build 生成 imagedefinitions.json
  EKS-->>Dev: 新版本 Pod 滚动更新完成
```

---

## Lab12 — Istio 服务网格

可选 Lab（面向微服务数量 ≥ 50 的中大型客户场景）。中国区镜像全部转存到 ECR；Ingress Gateway 强制改为 `internal` NLB。两个关键点决定了验证方式：① 验证用的临时 Pod 不能建在开了 `istio-injection=enabled` 的命名空间，否则自己也被注入 sidecar；② 集群内验证一律改用 Ingress Gateway 的 ClusterIP（绕开 DNS 解析和跨可用区负载均衡问题）。金丝雀发布路径为 100%v1 → 90/10 → 50/50 → 100%v3，比 Global 版本多一级过渡。可观测性额外部署 Kiali 与 Jaeger。

```mermaid
flowchart TB
  ECR["ECR 中国\n(istio/pilot、examples-bookinfo-*、\njaeger/kiali/prometheus 镜像)"]
  IstioD["istiod 控制面\n(hub=ECR中国)"]
  GatewayInternal["istio-ingressgateway\n(annotation: aws-load-balancer-internal=true)"]

  subgraph Bookinfo["namespace bookinfo (istio-injection=enabled)"]
    ProductPage["productpage-v1"]
    Details["details-v1"]
    ReviewsV1["reviews-v1"]
    ReviewsV3["reviews-v3"]
    Ratings["ratings-v1"]
  end

  VS["VirtualService reviews\n(金丝雀: 100%v1→90/10→50/50→100%v3)"]
  PeerAuth["PeerAuthentication STRICT"]
  Authz["AuthorizationPolicy\n(仅 productpage SA 可调用 reviews)"]
  Kiali["Kiali\n(服务拓扑 + mTLS锁图标 + 实时RPS)"]
  Jaeger["Jaeger\n(productpage→details+reviews→ratings 链路追踪)"]
  Verify["验证 Pod\n(不可建在 bookinfo ns，走 ClusterIP 而非 NLB 主机名)"]

  ECR --> IstioD --> GatewayInternal --> ProductPage
  ProductPage --> Details
  ProductPage --> VS --> ReviewsV1
  VS --> ReviewsV3
  ReviewsV1 --> Ratings
  ReviewsV3 --> Ratings
  PeerAuth -.mTLS强制.-> Bookinfo
  Authz -.服务间授权.-> Bookinfo
  Bookinfo -.拓扑上报.-> Kiali
  Bookinfo -.trace上报.-> Jaeger
  Verify -->|"ClusterIP直连(绕开DNS/跨AZ问题)"| GatewayInternal
```
