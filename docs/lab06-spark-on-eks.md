# Lab06 — Spark on EKS

#### 更新时间: 2026-07-17
#### 基于EKS版本: EKS 1.35

## 实验简介

演示在 EKS 上跑 Spark 数据处理任务，覆盖 Spark Operator（开源方案）、EMR on EKS（AWS 托管方案）、Karpenter Spot 跑 Executor。

> **中国区适配**：
> - EMR on EKS 在 cn-northwest-1 已可用
> - Spark Operator / Spark 镜像已转存到 ECR 中国（当前预置的是 `spark-py:4.1.2-python3`，执行时确认镜像 tag 与本文档一致）
> - 数据样本（NYC Taxi 结构）由操作机本地生成（原方案依赖的 S3 工具桶 `eks-tools-transfer-${ACCOUNT_ID}` 实测**不存在**，见步骤1）
> - **已知限制**：官方 `spark-py` 镜像不含 `hadoop-aws` jar，`mainApplicationFile` 若直接写 `s3a://` 路径会在 spark-submit 下载脚本阶段就报 `ClassNotFoundException`。Spark Operator 部分的 workaround：脚本用 ConfigMap 挂载成 `local://` 路径，数据用 `spark.range()` 集群内生成；S3 读写的完整验证放到下面"EMR on EKS"部分（EMR 用托管镜像，自带 S3 访问能力，不受此限制）
> - `hadoopConf` 里 `fs.s3a.aws.credentials.provider` 已按 IRSA 场景把 `WebIdentityTokenCredentialsProvider` 放在最前面——EMR-on-EKS/Spark 作业走 IRSA（`AWS_WEB_IDENTITY_TOKEN_FILE`），默认凭证链不含这个 Provider，不显式配置会报 `NoAuthWithAWSException`（完全无凭证，不是权限被拒）
> - 复用 Lab01 的共享集群 `adv-shared`（Auto Mode）：第三部分 Karpenter Spot NodePool 需要用 Auto Mode 的 `eks.amazonaws.com`/`NodeClass`，不是开源 Karpenter 的 `karpenter.k8s.aws`/`EC2NodeClass`（已在下方步骤更正，已实测确认可用）

**实验目标：**
- 掌握 Spark Operator（开源）与 EMR on EKS（托管）两条数据处理路径在中国区的部署方式
- 理解 IRSA 与 Pod Identity 在 S3 凭证配置上的差异
- 能够独立完成从数据准备到 Driver 常驻 + Executor Spot 的成本优化闭环

**实验流程：**
1. 准备数据与脚本
2. 安装 Spark Operator
3. 创建 Spark ServiceAccount 与 RBAC
4. 配置 Pod Identity 访问 S3
5. 提交 Spark 作业
6. 观察作业
7. 创建 EMR Virtual Cluster
8. 创建 EMR 执行 Role
9. 提交 EMR Spark 作业
10. 创建 Spark Spot NodePool
11. 提交 Driver 普通节点 + Executor Spot 的作业

**预计 AI 执行时长：** 25-30 分钟

## 前提条件

- Lab01 已完成，共享集群 `adv-shared` 存活
- Spark Operator / Spark 镜像已转存到 ECR 中国
- EMR Job Execution Role 除了 S3 读权限外，还需要 `s3:PutObject`/`s3:DeleteObject`（作业以 `overwrite` 模式写输出，需要能先删除旧文件）

```bash
export AWS_PROFILE=cn
export AWS_REGION=cn-northwest-1
export AWS_DEFAULT_REGION=cn-northwest-1
export CLUSTER_NAME=adv-shared
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REGISTRY=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn
export SPARK_NS=spark-jobs
export DATA_BUCKET=spark-data-${ACCOUNT_ID}
export EMR_VIRTUAL_CLUSTER=spark-on-eks-vc

aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}
```

---

## 步骤

### 1. 准备数据与脚本

S3 工具桶 `eks-tools-transfer-${ACCOUNT_ID}` 实测不存在（`aws s3 ls` 确认账号下没有这个桶），改为操作机本地用 pandas/pyarrow 生成结构相同的合成数据：

```bash
aws s3 mb s3://${DATA_BUCKET} --region ${AWS_REGION} 2>/dev/null || true

python3 <<'PYEOF'
import pandas as pd
import numpy as np
np.random.seed(42)
n = 200000
df = pd.DataFrame({
    "passenger_count": np.random.randint(1, 5, n).astype("int64"),
    "trip_distance": np.round(np.random.exponential(3.0, n), 2).astype("float64"),
    "fare_amount": np.round(np.random.exponential(15.0, n), 2).astype("float64"),
    "pickup_borough": np.random.choice(["Manhattan","Brooklyn","Queens","Bronx","StatenIsland"], n),
})
df.to_parquet("/tmp/nyc-taxi-sample.parquet", index=False)
PYEOF

aws s3 cp /tmp/nyc-taxi-sample.parquet s3://${DATA_BUCKET}/input/nyc-taxi.parquet --region ${AWS_REGION}

cat <<'EOF' > /tmp/taxi_analysis.py
import sys
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, count, avg

spark = SparkSession.builder.appName("NYCTaxiAnalysis").getOrCreate()
df = spark.read.parquet(sys.argv[1])
df.printSchema()
print(f"Total trips: {df.count()}")
result = df.groupBy("passenger_count").agg(
    count("*").alias("trips"),
    avg("trip_distance").alias("avg_distance")
)
result.show()
result.write.mode("overwrite").parquet(sys.argv[2])
spark.stop()
EOF

aws s3 cp /tmp/taxi_analysis.py s3://${DATA_BUCKET}/scripts/taxi_analysis.py --region ${AWS_REGION}

# Spark Operator 路径专用：spark-py 镜像缺 hadoop-aws，mainApplicationFile 不能是 s3a://
# （连脚本本身都下载不了），额外准备一份读集群内合成数据的版本，用 ConfigMap 挂载
cat <<'EOF' > /tmp/taxi_analysis_synthetic.py
from pyspark.sql import SparkSession
from pyspark.sql.functions import count, avg, rand, floor

spark = SparkSession.builder.appName("NYCTaxiAnalysisSynthetic").getOrCreate()
df = spark.range(0, 200000).withColumn("passenger_count", (floor(rand() * 4) + 1).cast("int")) \
    .withColumn("trip_distance", rand() * 20)
print(f"Total trips: {df.count()}")
result = df.groupBy("passenger_count").agg(
    count("*").alias("trips"),
    avg("trip_distance").alias("avg_distance")
)
result.show()
spark.stop()
EOF
```

**预期输出**：数据和脚本上传成功，`aws s3 ls s3://${DATA_BUCKET}/input/` 和 `s3://${DATA_BUCKET}/scripts/` 均能看到对应文件。

### 第一部分：Spark Operator on EKS

#### 2. 安装 Spark Operator

```bash
helm repo add spark-operator https://kubeflow.github.io/spark-operator
helm repo update

kubectl create namespace spark-operator 2>/dev/null || true
kubectl create namespace ${SPARK_NS} 2>/dev/null || true

helm upgrade --install spark-operator spark-operator/spark-operator \
  --namespace spark-operator \
  --version 2.0.2 \
  --set image.registry=${ECR_REGISTRY} \
  --set image.repository=kubeflow/spark-operator \
  --set image.tag=2.0.2 \
  --set webhook.enable=true \
  --set spark.jobNamespaces[0]=${SPARK_NS}

kubectl rollout status deployment/spark-operator-controller -n spark-operator --timeout=120s
```

**预期输出**：`deployment "spark-operator-controller" successfully rolled out`。

#### 3. 创建 Spark ServiceAccount 与 RBAC

```bash
kubectl -n ${SPARK_NS} create serviceaccount spark 2>/dev/null || true

kubectl -n ${SPARK_NS} create rolebinding spark-role \
  --clusterrole=edit \
  --serviceaccount=${SPARK_NS}:spark 2>/dev/null || true
```

**预期输出**：`serviceaccount/spark created`、`rolebinding.rbac.authorization.k8s.io/spark-role created`。

#### 4. 配置 Pod Identity 访问 S3

```bash
cat <<EOF > /tmp/spark-s3-policy.json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws-cn:s3:::${DATA_BUCKET}",
      "arn:aws-cn:s3:::${DATA_BUCKET}/*"
    ]
  }]
}
EOF

aws iam create-policy \
  --policy-name spark-s3-policy \
  --policy-document file:///tmp/spark-s3-policy.json 2>&1 || echo "(policy 已存在)"

eksctl create podidentityassociation \
  --cluster ${CLUSTER_NAME} \
  --namespace ${SPARK_NS} \
  --service-account-name spark \
  --permission-policy-arns arn:aws-cn:iam::${ACCOUNT_ID}:policy/spark-s3-policy \
  --role-name spark-s3-role \
  --region ${AWS_REGION}
```

**预期输出**：`created pod identity association for service account "spark" in namespace "spark-jobs"`。

#### 5. 提交 Spark 作业

**实测确认**：`mainApplicationFile: s3a://...` 会在 spark-submit 阶段（用户代码运行之前）就因缺少 `hadoop-aws` 报 `ClassNotFoundException: Class org.apache.hadoop.fs.s3a.S3AFileSystem not found` 而失败，不只是数据读写受影响。用 ConfigMap 把步骤1准备的 `taxi_analysis_synthetic.py` 挂载成 Pod 内的 `local://` 路径规避：

```bash
kubectl -n ${SPARK_NS} create configmap taxi-script --from-file=taxi_analysis_synthetic.py=/tmp/taxi_analysis_synthetic.py

cat <<EOF | kubectl apply -f -
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: nyc-taxi-analysis
  namespace: ${SPARK_NS}
spec:
  type: Python
  pythonVersion: "3"
  mode: cluster
  image: ${ECR_REGISTRY}/spark-py:4.1.2-python3
  imagePullPolicy: Always
  mainApplicationFile: local:///opt/spark/scripts/taxi_analysis_synthetic.py
  sparkVersion: 3.5.3
  driver:
    cores: 1
    memory: "1g"
    serviceAccount: spark
    volumeMounts:
    - name: script-vol
      mountPath: /opt/spark/scripts
  executor:
    cores: 1
    instances: 2
    memory: "1g"
    serviceAccount: spark
  volumes:
  - name: script-vol
    configMap:
      name: taxi-script
  restartPolicy:
    type: Never
EOF
```

**预期输出**：`sparkapplication.sparkoperator.k8s.io/nyc-taxi-analysis created`。这条路径只验证调度/执行链路（合成数据，集群内生成），不涉及 S3 读写——S3 读写的完整验证交给第二部分 EMR on EKS（已实测确认可用，见步骤9）。

> 如果确实需要验证 Spark Operator 直接读写 S3（非本 Lab 重点），需要在镜像里补装 `hadoop-aws`+`aws-java-sdk-bundle` jar 自建镜像，官方 `spark-py:4.1.2-python3` 不含这两个 jar。

#### 6. 观察作业

```bash
kubectl get sparkapplication -n ${SPARK_NS} -w

kubectl get pods -n ${SPARK_NS}

DRIVER_POD=$(kubectl get pods -n ${SPARK_NS} -l spark-role=driver -o jsonpath='{.items[0].metadata.name}')
kubectl logs -n ${SPARK_NS} ${DRIVER_POD} --tail=50
```

**预期输出**：`APPLICATION STATE` 从 `SUBMITTED` → `RUNNING` → `COMPLETED`；`kubectl get pods` 显示 1 个 driver + 2 个 executor；driver 日志末尾能看到 `groupBy(passenger_count)` 的结果表格。

### 第二部分：EMR on EKS

#### 7. 创建 EMR Virtual Cluster

```bash
aws emr-containers create-virtual-cluster \
  --name ${EMR_VIRTUAL_CLUSTER} \
  --container-provider '{
    "id": "'${CLUSTER_NAME}'",
    "type": "EKS",
    "info": {
      "eksInfo": {
        "namespace": "'${SPARK_NS}'"
      }
    }
  }' \
  --region ${AWS_REGION}

export VC_ID=$(aws emr-containers list-virtual-clusters \
  --query "virtualClusters[?name=='${EMR_VIRTUAL_CLUSTER}'].id" \
  --output text \
  --region ${AWS_REGION})
```

**预期输出**：`VC_ID` 取到非空的 virtual cluster ID。

#### 8. 创建 EMR 执行 Role

```bash
cat <<EOF > /tmp/emr-trust.json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "elasticmapreduce.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name EMRContainersJobExecutionRole \
  --assume-role-policy-document file:///tmp/emr-trust.json 2>&1 || true

aws iam attach-role-policy \
  --role-name EMRContainersJobExecutionRole \
  --policy-arn arn:aws-cn:iam::${ACCOUNT_ID}:policy/spark-s3-policy

aws emr-containers update-role-trust-policy \
  --cluster-name ${CLUSTER_NAME} \
  --namespace ${SPARK_NS} \
  --role-name EMRContainersJobExecutionRole \
  --region ${AWS_REGION}

export EMR_ROLE_ARN=arn:aws-cn:iam::${ACCOUNT_ID}:role/EMRContainersJobExecutionRole
```

**预期输出**：Role 创建/策略挂载成功，`update-role-trust-policy` 无报错。`spark-s3-policy` 已包含 `PutObject`/`DeleteObject`（步骤4），而不仅是只读——作业用 `overwrite` 模式写输出需要先删除旧文件，只给只读权限会在这一步报 `AccessDenied(DeleteObject)`。

#### 9. 提交 EMR Spark 作业

```bash
aws emr-containers start-job-run \
  --virtual-cluster-id ${VC_ID} \
  --name nyc-taxi-emr \
  --execution-role-arn ${EMR_ROLE_ARN} \
  --release-label emr-7.5.0-latest \
  --job-driver '{
    "sparkSubmitJobDriver": {
      "entryPoint": "s3://'${DATA_BUCKET}'/scripts/taxi_analysis.py",
      "entryPointArguments": [
        "s3://'${DATA_BUCKET}'/input/nyc-taxi.parquet",
        "s3://'${DATA_BUCKET}'/output-emr/"
      ],
      "sparkSubmitParameters": "--conf spark.executor.instances=2 --conf spark.executor.memory=1g --conf spark.driver.memory=1g --conf spark.hadoop.fs.s3a.aws.credentials.provider=com.amazonaws.auth.WebIdentityTokenCredentialsProvider"
    }
  }' \
  --region ${AWS_REGION}

aws emr-containers list-job-runs --virtual-cluster-id ${VC_ID} --region ${AWS_REGION}

kubectl get pods -n ${SPARK_NS} -w
```

**预期输出**：`list-job-runs` 里作业状态从 `PENDING`/`RUNNING` 最终变为 `COMPLETED`。`sparkSubmitParameters` 显式加了 `spark.hadoop.fs.s3a.aws.credentials.provider=...WebIdentityTokenCredentialsProvider`——EMR-on-EKS 作业走 IRSA 机制，hadoop-aws 默认凭证提供链不含能读 `AWS_WEB_IDENTITY_TOKEN_FILE` 的 Provider，不显式配置会报 `NoAuthWithAWSException`。

### 第三部分：Karpenter Spot 跑 Executor

> Driver 不放 Spot（中断会导致整个作业失败）。Executor 可放 Spot（Spark 自动重试丢失的 task）。`adv-shared` 是 Auto Mode 集群，NodePool 用内置 `eks.amazonaws.com`/`NodeClass`，不是开源 Karpenter 的 `karpenter.k8s.aws`/`EC2NodeClass`。

#### 10. 创建 Spark Spot NodePool

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spark-spot
spec:
  template:
    metadata:
      labels:
        workload: spark
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["m5.large", "m5.xlarge", "m5a.large", "m5a.xlarge", "r5.large", "r5.xlarge"]
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      taints:
        - key: spark-spot
          value: "true"
          effect: NoSchedule
      expireAfter: 24h
  limits:
    cpu: "200"
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 30s
EOF
```

**预期输出**：`nodepool.karpenter.sh/spark-spot created`。

#### 11. 提交 Driver 普通节点 + Executor Spot 的作业

```bash
cat <<EOF | kubectl apply -f -
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: nyc-taxi-spot
  namespace: ${SPARK_NS}
spec:
  type: Python
  pythonVersion: "3"
  mode: cluster
  image: ${ECR_REGISTRY}/spark-py:4.1.2-python3
  mainApplicationFile: s3a://${DATA_BUCKET}/scripts/taxi_analysis.py
  sparkVersion: 3.5.3
  hadoopConf:
    "fs.s3a.aws.credentials.provider": "com.amazonaws.auth.WebIdentityTokenCredentialsProvider,com.amazonaws.auth.ContainerCredentialsProvider,com.amazonaws.auth.InstanceProfileCredentialsProvider"
    "fs.s3a.endpoint": "s3.${AWS_REGION}.amazonaws.com.cn"
  arguments:
    - "s3a://${DATA_BUCKET}/input/nyc-taxi.parquet"
    - "s3a://${DATA_BUCKET}/output-spot/"
  driver:
    cores: 1
    memory: "1g"
    serviceAccount: spark
  executor:
    cores: 1
    instances: 4
    memory: "1g"
    serviceAccount: spark
    nodeSelector:
      workload: spark
    tolerations:
    - key: spark-spot
      operator: Equal
      value: "true"
      effect: NoSchedule
  restartPolicy:
    type: Never
  dynamicAllocation:
    enabled: false
EOF

kubectl get nodes -L karpenter.sh/capacity-type -L workload
```

**预期输出**：约 1-2 分钟内出现若干 `CAPACITY-TYPE=spot`、`WORKLOAD=spark` 的新节点，4 个 executor Pod 调度到这些 Spot 节点上。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 所有核心资源创建成功且状态正常
- [ ] 验证检查点中的所有命令返回预期结果
- [ ] 理解 IRSA 与 Pod Identity 在 S3 凭证配置上的差异

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get sparkapplication nyc-taxi-analysis -n ${SPARK_NS} -o jsonpath='{.status.applicationState.state}'` | `COMPLETED`（或已知限制下的合成数据变体） |
| 2 | `aws emr-containers list-job-runs --virtual-cluster-id ${VC_ID} --region ${AWS_REGION} --query 'jobRuns[0].state' --output text` | `COMPLETED` |
| 3 | `kubectl get nodes -L karpenter.sh/capacity-type --no-headers \| grep -c spot` | 大于等于 `1` |
| 4 | `kubectl get pods -n ${SPARK_NS} -l spark-role=executor --no-headers \| grep Running \| wc -l \| tr -d ' '` | 大于 `0`（**需在步骤11提交作业后的几十秒运行窗口内执行**——合成数据量小，作业跑得很快，executor Pod 完成后会被迅速回收，检查晚了会看到 `0`，这不代表作业失败，可改查 `kubectl get sparkapplication nyc-taxi-spot -o jsonpath='{.status.applicationState.state}'` 确认 `COMPLETED`） |

---

## 实验总结

本 Lab 对比了 Spark Operator（开源自建）与 EMR on EKS（AWS 托管）两条路径在中国区的运行方式，并演示了 Driver 常驻节点 + Executor Spot 的成本优化模式，同时记录了 IRSA 凭证 Provider 配置这一容易被忽略的必需项。实测确认官方 `spark-py` 镜像缺 `hadoop-aws` 的影响范围比预想更广（连 `mainApplicationFile: s3a://` 本身都会在 spark-submit 阶段失败），而 EMR on EKS 路径用真实 S3 数据完整跑通（读输入、写输出、`_SUCCESS` 标记文件全部确认存在），印证了"开源自建需要自己处理依赖，托管服务开箱即用"这一权衡。Lab07 将学习 Gateway API 和 LB Controller 流量入口进阶。

---

## 清理

```bash
kubectl delete sparkapplication --all -n ${SPARK_NS}

aws emr-containers delete-virtual-cluster --id ${VC_ID} --region ${AWS_REGION}

helm uninstall spark-operator -n spark-operator
kubectl delete namespace spark-operator
kubectl delete namespace ${SPARK_NS}

kubectl delete nodepool spark-spot

eksctl delete podidentityassociation \
  --cluster ${CLUSTER_NAME} \
  --namespace ${SPARK_NS} \
  --service-account-name spark \
  --region ${AWS_REGION}

aws iam delete-role --role-name spark-s3-role 2>&1 || true
aws iam detach-role-policy \
  --role-name EMRContainersJobExecutionRole \
  --policy-arn arn:aws-cn:iam::${ACCOUNT_ID}:policy/spark-s3-policy 2>&1 || true
aws iam delete-role --role-name EMRContainersJobExecutionRole 2>&1 || true
aws iam delete-policy --policy-arn arn:aws-cn:iam::${ACCOUNT_ID}:policy/spark-s3-policy 2>&1 || true

aws s3 rb s3://${DATA_BUCKET} --force --region ${AWS_REGION}
```

**预期输出**：Spark 相关命名空间、EMR virtual cluster、NodePool、IAM 角色/策略全部清除确认无残留。共享集群 `adv-shared` 不删除，供后续 Lab 复用。
