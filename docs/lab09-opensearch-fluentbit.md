# Lab09 — OpenSearch 和 Fluent Bit 日志平台

#### 更新时间: 2026-07-16
#### 基于EKS版本: EKS 1.35

## 实验简介

使用 Fluent Bit（`aws-for-fluent-bit`）采集 Pod 日志并发送到 Amazon OpenSearch Service。

> **中国区适配**：
> - `eks/aws-for-fluent-bit` Helm chart 依赖 `https://aws.github.io/eks-charts`，中国区可能无法访问，改用直接 `kubectl apply` DaemonSet YAML + ECR Public 镜像 `public.ecr.aws/aws-observability/aws-for-fluent-bit:2.32.4`
> - AL2023 节点 containerd 输出为 CRI 格式，Parser 必须用 `cri`（不是 `docker`）
> - 中国区 OpenSearch 端点域名格式带 `.cn` 后缀：`search-<name>-<hash>.cn-northwest-1.es.amazonaws.com.cn`
> - **未在中国区验证的已知风险（来自 global-eks-advance 同名 Lab 的实测发现）**：Fluent Bit 可能表面一切正常（Pod Running、无 403、HTTP 200），但如果 `kubernetes.labels.app`（带点）和其他不带点的标签字段类型冲突，会触发 OpenSearch 的 `mapper_parsing_exception`，日志实际写入全部失败但不易发现，需要开 `Trace_Error On` 才能看到。建议部署 Fluent Bit 时直接加 `Replace_Dots On`，并在验证检查点里确认索引真的有数据（不能只看 Pod 状态）。

**实验目标：**
- 掌握中国区 OpenSearch + Fluent Bit 日志采集链路的部署方式（绕开 `eks-charts` 依赖）
- 理解 AL2023 CRI Parser 适配，以及"看似成功实则写入失败"这类隐蔽故障模式
- 能够独立完成从 IAM Policy 配置到确认索引真实写入数据的完整闭环

**实验流程：**
1. 创建 Fluent Bit IAM Policy 和 Pod Identity 关联
2. 创建 OpenSearch 域
3. 映射 Fluent Bit IAM Role 到 OpenSearch 角色
4. 部署 Fluent Bit

**预计 AI 执行时长：** 20-25 分钟（含 OpenSearch 域创建约 10-15 分钟等待）

## 前提条件

- 已安装 `eks-pod-identity-agent`（Quickstart Demo01）
- OpenSearch 域访问密码需自定义（大小写字母+数字+特殊字符，至少8位）
- **成本警告**：OpenSearch 域持续计费，演示结束后立即删除

```bash
export AWS_PROFILE=cn
export AWS_REGION=cn-northwest-1
export AWS_DEFAULT_REGION=cn-northwest-1
export CLUSTER_NAME=demo
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ES_DOMAIN_NAME=test-logging
export ES_DOMAIN_USER=test
export ES_DOMAIN_PASSWORD="<自定义密码，需含大小写字母+数字+特殊字符，至少8位>"
```

---

## 步骤

### 1. 创建 Fluent Bit IAM Policy 和 Pod Identity 关联

```bash
cat <<EOF > /tmp/fluent-bit-policy.json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["es:ESHttp*"],
    "Resource": "arn:aws-cn:es:${AWS_REGION}:${ACCOUNT_ID}:domain/${ES_DOMAIN_NAME}"
  }]
}
EOF

aws iam create-policy \
  --policy-name fluent-bit-policy \
  --policy-document file:///tmp/fluent-bit-policy.json

kubectl create namespace logging

eksctl create podidentityassociation \
  --cluster ${CLUSTER_NAME} \
  --namespace logging \
  --service-account-name fluent-bit \
  --permission-policy-arns arn:aws-cn:iam::${ACCOUNT_ID}:policy/fluent-bit-policy \
  --role-name demo-fluent-bit \
  --create-service-account \
  --region ${AWS_REGION}
```

**预期输出**：`created pod identity association for service account "fluent-bit" in namespace "logging"`。

### 2. 创建 OpenSearch 域

```bash
cat <<EOF | envsubst > /tmp/es_domain.json
{
  "DomainName": "${ES_DOMAIN_NAME}",
  "EngineVersion": "OpenSearch_3.5",
  "ClusterConfig": {
    "InstanceType": "t3.medium.search",
    "InstanceCount": 1,
    "DedicatedMasterEnabled": false,
    "ZoneAwarenessEnabled": false
  },
  "EBSOptions": { "EBSEnabled": true, "VolumeType": "gp3", "VolumeSize": 20 },
  "NodeToNodeEncryptionOptions": { "Enabled": true },
  "EncryptionAtRestOptions": { "Enabled": true },
  "DomainEndpointOptions": { "EnforceHTTPS": true },
  "AdvancedSecurityOptions": {
    "Enabled": true,
    "InternalUserDatabaseEnabled": true,
    "MasterUserOptions": {
      "MasterUserName": "${ES_DOMAIN_USER}",
      "MasterUserPassword": "${ES_DOMAIN_PASSWORD}"
    }
  },
  "AccessPolicies": "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"AWS\":\"*\"},\"Action\":\"es:*\",\"Resource\":\"arn:aws-cn:es:${AWS_REGION}:${ACCOUNT_ID}:domain/${ES_DOMAIN_NAME}/*\"}]}"
}
EOF

aws opensearch create-domain --cli-input-json file:///tmp/es_domain.json

# 等待域就绪（约 10–15 分钟）
until ES_ENDPOINT=$(aws opensearch describe-domain \
  --domain-name ${ES_DOMAIN_NAME} --region ${AWS_REGION} \
  --query "DomainStatus.Endpoint" --output text) && [ "${ES_ENDPOINT}" != "None" ]; do
  echo "Provisioning..."; sleep 30
done
echo "OpenSearch endpoint: ${ES_ENDPOINT}"
```

**预期输出**：约 10-15 分钟后打印 `OpenSearch endpoint: search-test-logging-<hash>.cn-northwest-1.es.amazonaws.com.cn`（带 `.cn` 后缀）。域的 `AccessPolicies` 用 IAM Principal 控制（不是网络层 0.0.0.0/0 入站规则），细粒度访问由 fine-grained access control + master user 承担。如需更严格的网络隔离，可改用 VPC-based OpenSearch 域。

### 3. 映射 Fluent Bit IAM Role 到 OpenSearch 角色

```bash
export FLUENTBIT_ROLE=$(eksctl get podidentityassociation \
  --cluster ${CLUSTER_NAME} --namespace logging \
  --service-account-name fluent-bit --region ${AWS_REGION} \
  -o json | jq -r '.[].RoleARN')

curl -sS -u "${ES_DOMAIN_USER}:${ES_DOMAIN_PASSWORD}" \
  -X PATCH \
  "https://${ES_ENDPOINT}/_plugins/_security/api/rolesmapping/all_access?pretty" \
  -H 'Content-Type: application/json' \
  -d "[{\"op\":\"add\",\"path\":\"/backend_roles\",\"value\":[\"${FLUENTBIT_ROLE}\"]}]"
```

**预期输出**：`{"status" : "OK", "message" : "'all_access' updated."}`。

### 4. 部署 Fluent Bit

中国区改用直接 kubectl 部署 DaemonSet（避免依赖 `aws.github.io/eks-charts`）：

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: aws-for-fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: aws-for-fluent-bit
  template:
    metadata:
      labels:
        app: aws-for-fluent-bit
    spec:
      serviceAccountName: fluent-bit
      containers:
      - name: fluent-bit
        image: public.ecr.aws/aws-observability/aws-for-fluent-bit:2.32.4
        env:
        - name: AWS_REGION
          value: ${AWS_REGION}
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc/
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Parsers_File parsers.conf

    [INPUT]
        Name tail
        Path /var/log/containers/*.log
        Parser cri
        Tag kube.*
        Refresh_Interval 5

    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On

    [OUTPUT]
        Name es
        Match kube.*
        Host ${ES_ENDPOINT}
        Port 443
        TLS On
        AWS_Auth On
        AWS_Region ${AWS_REGION}
        Replace_Dots On
        Index fluent-bit
  parsers.conf: |
    [PARSER]
        Name cri
        Format regex
        Regex ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[FP]) (?<log>.*)$
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
EOF

kubectl -n logging rollout status daemonset/aws-for-fluent-bit
```

**预期输出**：`daemon set "aws-for-fluent-bit" successfully rolled out`。`Parser cri` 是 AL2023 节点必须的配置（containerd 输出 CRI 格式，不是 docker JSON 格式）。`Replace_Dots On` 避免 `kubernetes.labels.app`（带点）字段与其他标签字段类型冲突导致 OpenSearch 端 `mapper_parsing_exception`（这个问题表面看不出来——Pod Running、无 403、HTTP 200，但日志实际写入会静默失败，只有开 `Trace_Error On` 才能看到）。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 所有核心资源创建成功且状态正常
- [ ] 验证检查点中的所有命令返回预期结果
- [ ] **索引里真的有数据**（不能只看 Pod 状态判断成功——理解为什么这一点必须单独验证）

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws opensearch describe-domain --domain-name ${ES_DOMAIN_NAME} --region ${AWS_REGION} --query 'DomainStatus.Processing' --output text` | `False` |
| 2 | `kubectl -n logging get pods -l app=aws-for-fluent-bit --no-headers \| grep Running \| wc -l \| tr -d ' '` | 大于 `0` |
| 3 | `curl -sS -u "${ES_DOMAIN_USER}:${ES_DOMAIN_PASSWORD}" "https://${ES_ENDPOINT}/fluent-bit*/_count" \| jq -r '.count'` | 大于 `0`，且随时间持续增长（不是只看 Pod Running） |

登录 OpenSearch Dashboards（`https://${ES_ENDPOINT}/_dashboards/`）创建 Index Pattern：
1. **Stack Management** → **Index Patterns** → **Create index pattern**
2. Pattern 填 `*fluent-bit*`，时间字段选 `@timestamp`
3. **Discover** 查看日志

---

## 实验总结

本 Lab 验证了中国区 OpenSearch + Fluent Bit 日志采集链路，重点是绕开 `eks-charts` 依赖、CRI Parser 适配，以及"看似成功实则写入失败"这类隐蔽故障模式的排查方法（直接查文档计数，而不只看 Pod 状态）。Lab10 将学习 EKS 集群升级与维护。

---

## 清理

```bash
kubectl delete daemonset aws-for-fluent-bit -n logging
kubectl delete configmap fluent-bit-config -n logging

aws opensearch delete-domain --domain-name ${ES_DOMAIN_NAME}

eksctl delete podidentityassociation \
  --cluster ${CLUSTER_NAME} --namespace logging \
  --service-account-name fluent-bit --region ${AWS_REGION}

aws iam delete-policy \
  --policy-arn arn:aws-cn:iam::${ACCOUNT_ID}:policy/fluent-bit-policy

kubectl delete namespace logging
```

**预期输出**：OpenSearch 域进入删除流程（约 5-10 分钟完成），DaemonSet/命名空间/IAM 资源立即清除确认无残留。
