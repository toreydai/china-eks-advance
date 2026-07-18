# EKS Advance Workshop
## AWS 中国区 EKS 进阶动手实验集

基于 EKS 1.35 · Amazon Linux 2023 · 宁夏区（cn-northwest-1）· Prompt 驱动执行

> 本 workshop 是 [`gcr-eks-quickstart`](../gcr-eks-quickstart/README.md) 的进阶补充，也是 [`global-eks-advance`](https://github.com/toreydai/global-eks-advance) 的中国区对应版本。建议完成 QuickStart 基础档后再执行本 workshop。

---

## Lab 列表

### 1. 平台与多集群治理

- [Lab01 — EKS Auto Mode 深度演示](docs/lab01-eks-auto-mode.md)
- [Lab02 — 多集群管理（ArgoCD Hub-Spoke）](docs/lab02-multi-cluster-argocd.md)
- [Lab11 — Kyverno 策略治理](docs/lab11-kyverno.md)

### 2. 成本、流量入口与交付

- [Lab03 — Kubecost 成本优化](docs/lab03-kubecost.md)
- [Lab07 — Gateway API 和 AWS Load Balancer Controller 入口进阶](docs/lab07-gateway-api-lbc.md)
- [Lab08 — CodePipeline 进行 EKS CI/CD](docs/lab08-codepipeline-cicd.md)

### 3. AI、数据与日志平台

- [Lab05 — GenAI 推理服务（GPU + vLLM）](docs/lab05-genai-gpu-vllm.md)
- [Lab06 — Spark on EKS](docs/lab06-spark-on-eks.md)
- [Lab09 — OpenSearch 和 Fluent Bit 日志平台](docs/lab09-opensearch-fluentbit.md)

### 4. 运维维护

- [Lab10 — EKS 集群升级与维护](docs/lab10-cluster-upgrade.md)
- [Lab12 — Istio 服务网格](docs/lab12-istio.md)（可选，适用于微服务数量 ≥ 50 的中大型客户）

> **Lab04（AI Agent on EKS）在本 workshop 中不存在**：Bedrock 在 `cn-northwest-1`、`cn-north-1` 两个中国区 region 均无可用 endpoint，该场景暂无法在中国区演示。

---

## 与 QuickStart 的关系

建议先完成 [`gcr-eks-quickstart`](../gcr-eks-quickstart/README.md) 的基础档，再按客户场景选择本 workshop 的 Lab。QuickStart 负责 EKS 核心闭环（建集群、部署、监控、安全等 18 个 Demo）；本 workshop 负责 QuickStart 没覆盖或已迁出的进阶场景——平台治理、交付、日志、升级、安全治理、AI 和数据等专题，主线 Lab 全部与 QuickStart 不重合（Auto Mode / 多集群管理 / Kubecost / GenAI 推理 / Spark）。

以下内容原属于 QuickStart，因费用高、耗时长或 EKS 知识占比下降而迁入本 workshop：

| 原 QuickStart Demo | 新 Lab | 迁入理由 |
|-----------------|---------------|----------|
| Demo15 Gateway API | Lab07 | Gateway API 是入口模型进阶，不必占用 QuickStart 主线。 |
| Demo20 CodePipeline CI/CD | Lab08 | 跨多个 AWS 服务，EKS 知识占比下降，适合平台工程专题。 |
| Demo21 OpenSearch + Fluent Bit | Lab09 | OpenSearch 创建慢、有费用压力且需要 Dashboard 手动配置。 |
| Demo22 EKS 集群升级与维护 | Lab10 | 需要独立临时集群和完整升级流程，适合运维专项。 |
| Demo18 Kyverno 策略治理部分 | Lab11 | PSS 可作为 QuickStart 基础，Kyverno 更偏策略平台化。 |

---

## 与全球区版本（global-eks-advance）的差异

| 项目 | 中国区（本项目） | 全球区 |
|------|------------------|--------|
| IAM ARN | `arn:aws-cn:` | `arn:aws:` |
| ECR 域名 | `.amazonaws.com.cn` | `.amazonaws.com` |
| 工具 / Helm Chart | 从 S3 工具桶预置包，或本机 docker pull/tag/push 转存 ECR | 直接从 GitHub / Helm Hub 下载 |
| 镜像仓库 | `registry.k8s.io`/`ghcr.io`/`quay.io` 等受限，需私有 ECR | docker.io / quay.io / registry.k8s.io 均可访问 |
| Bedrock | 中国区不可用（无 Lab04） | us-east-1 可用 |
| OIDC issuer | `oidc.eks.${AWS_REGION}.amazonaws.com.cn` | `oidc.eks.${AWS_REGION}.amazonaws.com` |
| 安全组 | 硬性禁止 `0.0.0.0/0`，一律用 internal/port-forward | 视 Lab 而定 |

---

## 使用方式

1. 在此目录下打开 Claude Code，`CLAUDE.md` 自动加载中国区 EKS Advance 配置
2. 将对应 [`docs/`](docs/) 目录中的 Lab 文件内容粘贴到对话框，由 AI 自主执行
3. 每个 Lab 末尾均有清理要求；涉及 GPU、OpenSearch、多集群、EMR、Kubecost、Istio 的 Lab 应优先清理以避免持续计费

---

## 环境要求

| 工具 | 版本 |
|------|------|
| AWS CLI | 2.x（AL2023 预装） |
| eksctl | latest（Auto Mode 需 ≥ 0.196.0） |
| kubectl | v1.35 |
| Helm | 3.x |
| jq | latest |
| Docker | 24.x+（本机转存镜像用） |

操作机建议使用 Amazon Linux 2023 EC2，绑定具备 EKS、EC2、IAM、ECR、S3、CloudFormation、CodeBuild、CodePipeline、CodeCommit、OpenSearch、EMR Containers 操作权限的 IAM Role。全程使用 `AWS_PROFILE=cn`（中国区账号）。

---

## 特别说明

| Lab | 额外要求 |
|-----|---------|
| Lab02 多集群管理 | 需要至少 2 个 EKS 集群（建议提前预创建 Spoke 集群） |
| Lab05 GenAI 推理 | GPU 实例配额需提前申请；当前 `cn-northwest-1` offerings 显示 `g5.xlarge` 可用，未显示 `g6.xlarge` |
| Lab06 Spark on EKS | EMR on EKS 中国区可用；官方 Spark 镜像不含 hadoop-aws jar，见 `docs/lab06-spark-on-eks.md` 已知限制 |
| Lab09 OpenSearch | OpenSearch 域创建约 10-15 分钟且持续计费 |
| Lab12 Istio | 建议在共享集群或独立 namespace 中执行；可选 Lab，非主线必修 |

---

## License

MIT - see the [LICENSE](LICENSE) file for details.

## 免责声明

- 本项目仅供学习与技术参考，不构成生产部署方案。
- 运行过程中会创建 AWS 资源并产生费用，请在实验结束后及时清理。
- 作者不对因使用本项目产生的任何费用或损失承担责任。
- 本项目与 Amazon Web Services 无官方关联，相关服务的可用性与定价以 AWS 官方文档为准。
- 生产环境使用前请根据实际需求进行安全评估与调整。
