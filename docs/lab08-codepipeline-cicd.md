# Lab08 — CodePipeline 进行 EKS CI/CD

#### 更新时间: 2026-07-16
#### 基于EKS版本: EKS 1.35

## 实验简介

构建 CodeCommit → CodeBuild → EKS 的 CI/CD 流水线：代码推送后自动构建镜像并部署到集群。

> **中国区适配**：
> - 中国区无法访问 GitHub，示例代码（`main.go`/`Dockerfile`/`go.mod`/`buildspec.yml`/`k8s_deployment.yml`）需要提前准备好压缩包放到 S3，或在操作机上手动创建
> - CodeCommit HTTPS 推送需要 `git-remote-codecommit`，用 `codecommit::<region>://` 协议推送，不走标准 HTTPS credential helper
> - 操作机（AL2023）默认没有 `git`/`pip3`，需要先 `dnf install -y git` + `python3 -m ensurepip`
> - IAM 角色创建后立即使用可能报 `not authorized to perform: sts:AssumeRole`，需等待约 15 秒角色传播

**实验目标：**
- 掌握基于中国区原生服务（CodeCommit + CodeBuild + CodePipeline + ECR）的 CI/CD 闭环搭建
- 理解 CodeBuild 访问 EKS 集群所需的 IAM Access Entry 授权模型
- 能够独立完成从建 Pipeline 到验证自动触发重新部署的完整闭环

**实验流程：**
1. 创建 CodeBuild kubectl 角色并授权访问集群
2. 准备示例代码和 ECR/CodeCommit
3. 创建 CodeBuild 项目
4. 创建 CodePipeline
5. 触发新发布

**预计 AI 执行时长：** 15-20 分钟

## 前提条件

- 操作机已安装 AWS CLI 2.x，`AWS_PROFILE=cn`
- IAM 权限：IAM Role 创建、EKS Access Entry、ECR/CodeCommit/CodeBuild/CodePipeline 全套操作权限

```bash
export AWS_PROFILE=cn
export AWS_REGION=cn-northwest-1
export REGION=${AWS_REGION}
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export CLUSTER_NAME=demo
export S3_BUCKET=test-pipeline-${ACCOUNT_ID}
export TOOLS_BUCKET=eks-tools-transfer-${ACCOUNT_ID}

dnf install -y git
python3 -m ensurepip --upgrade && python3 -m pip install git-remote-codecommit
```

---

## 步骤

### 1. 创建 CodeBuild kubectl 角色并授权访问集群

```bash
TRUST="{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"AWS\":\"arn:aws-cn:iam::${ACCOUNT_ID}:root\"},\"Action\":\"sts:AssumeRole\"}]}"

aws iam create-role \
  --role-name EksWorkshopCodeBuildKubectlRole \
  --assume-role-policy-document "${TRUST}" \
  --query 'Role.Arn' --output text

aws iam put-role-policy \
  --role-name EksWorkshopCodeBuildKubectlRole \
  --policy-name eks-describe \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"eks:Describe*","Resource":"*"}]}'

# 用 Access Entry 授予集群管理权限
eksctl create accessentry \
  --cluster ${CLUSTER_NAME} \
  --principal-arn arn:aws-cn:iam::${ACCOUNT_ID}:role/EksWorkshopCodeBuildKubectlRole \
  --region ${REGION}

aws eks associate-access-policy \
  --cluster-name ${CLUSTER_NAME} \
  --principal-arn arn:aws-cn:iam::${ACCOUNT_ID}:role/EksWorkshopCodeBuildKubectlRole \
  --policy-arn arn:aws-cn:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster \
  --region ${REGION}

eksctl get accessentry --cluster ${CLUSTER_NAME} --region ${REGION}
```

**预期输出**：Role ARN 打印成功；`eksctl get accessentry` 列表中出现 `EksWorkshopCodeBuildKubectlRole` 的 access entry。

### 2. 准备示例代码和 ECR/CodeCommit

```bash
# 原始地址：https://github.com/rnzsgh/eks-workshop-sample-api-service-go.git
# 中国区无法访问 GitHub。优先尝试从 S3 中转下载；如 S3 上没有预置这个包，
# 在操作机上手动创建最小示例（main.go + Dockerfile + go.mod + buildspec.yml + k8s_deployment.yml）
aws s3 cp s3://${TOOLS_BUCKET}/eks-workshop-sample-api-service-go.tar.gz /tmp/ --region ${REGION} \
  && tar xzf /tmp/eks-workshop-sample-api-service-go.tar.gz \
  || echo "(S3 无预置包，需手动创建示例代码目录 eks-workshop-sample-api-service-go/)"

cd eks-workshop-sample-api-service-go

aws ecr create-repository --repository-name demo-codepipeline --region ${REGION}

aws codecommit create-repository \
  --repository-name eks-workshop-sample-api-service-go \
  --repository-description "EKS Workshop CodeCommit" \
  --region ${REGION}

git init
git add .
git commit -m 'initial'
# 用 git-remote-codecommit 协议推送，不走标准 HTTPS credential helper
git remote add origin codecommit::${REGION}://eks-workshop-sample-api-service-go
git branch -m main
git push -u origin main
cd ..
```

**预期输出**：ECR 仓库、CodeCommit 仓库创建成功；`git push` 输出 `* [new branch] main -> main`。

### 3. 创建 CodeBuild 项目

```bash
CODEBUILD_TRUST='{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"codebuild.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam create-role \
  --role-name EksWorkshopCodeBuild \
  --assume-role-policy-document "${CODEBUILD_TRUST}" \
  --query 'Role.Arn' --output text

cat <<'EOF' > /tmp/codebuild-policy.json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ecr:*", "logs:*", "s3:*", "codecommit:GitPull",
      "codebuild:*", "sts:AssumeRole", "iam:PassRole"
    ],
    "Resource": "*"
  }]
}
EOF

aws iam put-role-policy \
  --role-name EksWorkshopCodeBuild \
  --policy-name codebuild \
  --policy-document file:///tmp/codebuild-policy.json

# IAM 角色刚创建，等待传播后再用于 CodeBuild Project，否则报 not authorized to perform: sts:AssumeRole
sleep 15

aws codebuild create-project \
  --name demo-codebuild \
  --source "{\"type\":\"CODECOMMIT\",\"location\":\"https://git-codecommit.${REGION}.amazonaws.com.cn/v1/repos/eks-workshop-sample-api-service-go\"}" \
  --artifacts '{"type":"NO_ARTIFACTS"}' \
  --environment "{
    \"type\":\"LINUX_CONTAINER\",
    \"image\":\"aws/codebuild/standard:7.0\",
    \"computeType\":\"BUILD_GENERAL1_SMALL\",
    \"privilegedMode\":true,
    \"environmentVariables\":[
      {\"name\":\"REPOSITORY_URI\",\"value\":\"${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com.cn/demo-codepipeline\",\"type\":\"PLAINTEXT\"},
      {\"name\":\"REPOSITORY_NAME\",\"value\":\"eks-workshop-sample-api-service-go\",\"type\":\"PLAINTEXT\"},
      {\"name\":\"REPOSITORY_BRANCH\",\"value\":\"main\",\"type\":\"PLAINTEXT\"},
      {\"name\":\"EKS_CLUSTER_NAME\",\"value\":\"${CLUSTER_NAME}\",\"type\":\"PLAINTEXT\"},
      {\"name\":\"EKS_KUBECTL_ROLE_ARN\",\"value\":\"arn:aws-cn:iam::${ACCOUNT_ID}:role/EksWorkshopCodeBuildKubectlRole\",\"type\":\"PLAINTEXT\"}
    ]
  }" \
  --service-role arn:aws-cn:iam::${ACCOUNT_ID}:role/EksWorkshopCodeBuild \
  --region ${REGION}
```

**预期输出**：`aws codebuild create-project` 返回项目详情 JSON，`project.name` 为 `demo-codebuild`。`buildspec.yml` 的 `post_build` 阶段末尾必须生成 `imagedefinitions.json`（例如 `printf '[{"name":"...","imageUri":"..."}]' > imagedefinitions.json`），否则即使镜像构建成功，`artifacts` 阶段也会报 `UPLOAD_ARTIFACTS: no matching artifact paths found`。

### 4. 创建 CodePipeline

```bash
aws s3api create-bucket --bucket ${S3_BUCKET} --region ${REGION} \
  --create-bucket-configuration LocationConstraint=${REGION}

PIPELINE_TRUST='{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"codepipeline.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam create-role \
  --role-name EksWorkshopCodePipeline \
  --assume-role-policy-document "${PIPELINE_TRUST}" \
  --query 'Role.Arn' --output text

cat <<'EOF' > /tmp/codepipeline-policy.json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "codebuild:*", "codecommit:*", "s3:*", "ecr:*",
      "ecs:*", "iam:PassRole", "cloudwatch:*"
    ],
    "Resource": "*"
  }]
}
EOF

aws iam put-role-policy \
  --role-name EksWorkshopCodePipeline \
  --policy-name codepipeline \
  --policy-document file:///tmp/codepipeline-policy.json

cat <<EOF > /tmp/codepipeline.json
{
  "pipeline": {
    "name": "demo-codepipeline",
    "roleArn": "arn:aws-cn:iam::${ACCOUNT_ID}:role/EksWorkshopCodePipeline",
    "artifactStore": { "type": "S3", "location": "${S3_BUCKET}" },
    "stages": [
      {
        "name": "Source",
        "actions": [{
          "name": "Source",
          "actionTypeId": { "category":"Source","owner":"AWS","provider":"CodeCommit","version":"1" },
          "runOrder": 1,
          "configuration": {
            "BranchName": "main",
            "OutputArtifactFormat": "CODE_ZIP",
            "PollForSourceChanges": "true",
            "RepositoryName": "eks-workshop-sample-api-service-go"
          },
          "outputArtifacts": [{ "name": "SourceArtifact" }],
          "region": "${REGION}"
        }]
      },
      {
        "name": "Build",
        "actions": [{
          "name": "Build",
          "actionTypeId": { "category":"Build","owner":"AWS","provider":"CodeBuild","version":"1" },
          "runOrder": 1,
          "configuration": { "ProjectName": "demo-codebuild" },
          "inputArtifacts": [{ "name": "SourceArtifact" }],
          "outputArtifacts": [{ "name": "BuildArtifact" }],
          "region": "${REGION}"
        }]
      }
    ],
    "version": 1
  }
}
EOF

aws codepipeline create-pipeline --cli-input-json file:///tmp/codepipeline.json --region ${REGION}
```

**预期输出**：Pipeline 创建成功，控制台确认 Pipeline 执行中，Source 阶段很快 `Succeeded`。若 CodeBuild 因网络问题失败，选择重试即可。

### 5. 触发新发布

修改 `eks-workshop-sample-api-service-go/main.go` 第 18 行，然后推送：

```bash
cd eks-workshop-sample-api-service-go
# 修改: res := &response{Message: "Hello World from EKS"}
sed -i 's/Hello World/Hello World from EKS/' main.go
git add .
git commit -m 'update message'
git push origin main
cd ..
```

**预期输出**：`git push` 成功；Pipeline 自动触发新一轮执行，约 3-5 分钟完成构建和部署。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 所有核心资源创建成功且状态正常
- [ ] 验证检查点中的所有命令返回预期结果
- [ ] 理解 buildspec `imagedefinitions.json` 生成对 Pipeline Artifacts 阶段的必要性

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws codepipeline get-pipeline-state --name demo-codepipeline --region ${REGION} --query 'stageStates[?stageName==\`Source\`].latestExecution.status' --output text` | `Succeeded` |
| 2 | `aws codepipeline get-pipeline-state --name demo-codepipeline --region ${REGION} --query 'stageStates[?stageName==\`Build\`].latestExecution.status' --output text` | `Succeeded` |
| 3 | `aws ecr describe-images --repository-name demo-codepipeline --region ${REGION} --query 'length(imageDetails)' --output text` | 大于等于 `1` |
| 4 | `aws codepipeline list-pipeline-executions --name demo-codepipeline --region ${REGION} --query 'length(pipelineExecutionSummaries)' --output text` | 大于等于 `2`（初次 + 触发新发布） |

---

## 实验总结

本 Lab 验证了完全基于中国区原生服务（CodeCommit + CodeBuild + CodePipeline + ECR）的 CI/CD 闭环，覆盖了 IAM 角色传播延迟、CodeCommit 推送协议、buildspec artifacts 生成这几个容易踩坑的环节。Lab09 将学习 OpenSearch 和 Fluent Bit 日志平台。

---

## 清理

```bash
aws codepipeline delete-pipeline --name demo-codepipeline --region ${REGION}
aws codebuild delete-project --name demo-codebuild --region ${REGION}
aws codecommit delete-repository \
  --repository-name eks-workshop-sample-api-service-go --region ${REGION}
aws ecr delete-repository \
  --repository-name demo-codepipeline --force --region ${REGION}

aws s3 rm s3://${S3_BUCKET} --recursive
aws s3api delete-bucket --bucket ${S3_BUCKET} --region ${REGION}

aws eks disassociate-access-policy \
  --cluster-name ${CLUSTER_NAME} \
  --principal-arn arn:aws-cn:iam::${ACCOUNT_ID}:role/EksWorkshopCodeBuildKubectlRole \
  --policy-arn arn:aws-cn:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster --region ${REGION}
aws eks delete-access-entry \
  --cluster-name ${CLUSTER_NAME} \
  --principal-arn arn:aws-cn:iam::${ACCOUNT_ID}:role/EksWorkshopCodeBuildKubectlRole \
  --region ${REGION}

aws iam delete-role-policy --role-name EksWorkshopCodeBuild --policy-name codebuild
aws iam delete-role-policy --role-name EksWorkshopCodeBuildKubectlRole --policy-name eks-describe
aws iam delete-role-policy --role-name EksWorkshopCodePipeline --policy-name codepipeline
aws iam delete-role --role-name EksWorkshopCodeBuild
aws iam delete-role --role-name EksWorkshopCodeBuildKubectlRole
aws iam delete-role --role-name EksWorkshopCodePipeline
```

**预期输出**：Pipeline、CodeBuild 项目、CodeCommit/ECR 仓库、S3 桶、Access Entry、IAM 角色全部删除确认无残留。
