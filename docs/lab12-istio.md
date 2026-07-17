# Lab12 — Istio 服务网格

#### 更新时间: 2026-07-17
#### 基于EKS版本: EKS 1.35

## 实验简介

演示在 EKS 上部署 Istio 服务网格，覆盖流量管理（金丝雀发布）、安全（mTLS）、可观测（Kiali 拓扑 + Jaeger 追踪）。

> 本 workshop 中列为**可选 Lab**，适用于微服务数量 ≥ 50 的中大型客户场景；不属于主线 Lab（01-11）的必修范围。
>
> **中国区适配**：
> - Istio 官方镜像位于 `gcr.io`、`docker.io/istio`，需转存到私有 ECR；Bookinfo 示例应用镜像（`istio/examples-bookinfo-*`）也需要单独转存（这些是应用镜像，不属于 istio/pilot 等控制面镜像）
> - `istioctl` 二进制、Bookinfo/addons manifest 直接从操作机下载（原方案假设的 S3 工具桶 `eks-tools-transfer-${ACCOUNT_ID}` 实测**不存在**，见步骤1、3；操作机有完整外网直连，改为直接从 GitHub Releases 下载 Istio 官方 release 包，其中已包含 istioctl 二进制、Bookinfo manifest、addons manifest 三者，不需要三处分别下载）
> - `istioctl install --set profile=demo` 默认给 `istio-ingressgateway` Service 创建 LoadBalancer（AWS 上默认是 internet-facing），必须显式加 internal 注解，验证也相应改为内部访问（详见步骤2、3）
> - 连通性验证统一用 `busybox`（`wget`）而不是 `curlimages/curl`——该镜像在 ECR Public 的 Docker Official Images 命名空间下不存在（Lab01 已验证的同类问题）
> - **两处已修正**（步骤3、4）：① 验证用的测试 Pod 不能建在开了 `istio-injection=enabled` 的命名空间（如 `bookinfo`），否则自己也会被注入 istio-proxy sidecar，导致 `Connection refused`，容易误判为网络故障，应在 `default` 等未开注入的命名空间起测试 Pod。② Auto Mode 集群 VPC 的默认 DNS 解析不了内部 NLB 自动生成的主机名，且 NLB 默认关闭跨可用区负载均衡；集群内验证一律改用 `istio-ingressgateway` 的 **ClusterIP**，浏览器/公网场景才需要处理 NLB DNS/跨AZ

**实验目标：**
- 掌握 Istio 在中国区私有镜像仓库下的服务网格部署方式
- 理解流量管理（金丝雀）、mTLS 强制加密、服务间授权三大核心能力
- 能够独立完成从安装控制面到验证可观测性（Kiali/Jaeger）的完整闭环

**实验流程：**
1. 安装 istioctl
2. 安装 Istio 控制面
3. 部署 Bookinfo 示例
4. 流量管理：金丝雀发布
5. mTLS 自动加密
6. 服务间授权（AuthorizationPolicy）
7. 可观测性：Kiali + Jaeger

**预计 AI 执行时长：** 25-30 分钟

## 前提条件

- Istio 控制面镜像（`istio/pilot`、`istio/proxyv2`）、Bookinfo 应用镜像（`istio/examples-bookinfo-*` 6个）、可观测组件镜像（`jaegertracing/all-in-one`、`kiali/kiali`、`prom/prometheus`、`prometheus-operator/prometheus-config-reloader`）均已转存到 ECR 中国
- `istioctl` 二进制、Bookinfo/addons manifest 从 Istio 官方 GitHub Release 包直接获取（操作机有完整外网直连）

```bash
export AWS_PROFILE=cn
export AWS_REGION=cn-northwest-1
export AWS_DEFAULT_REGION=cn-northwest-1
export CLUSTER_NAME=adv-shared
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REGISTRY=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn
export ISTIO_VERSION=1.24.2
export ISTIO_HUB=${ECR_REGISTRY}/istio

aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}
```

---

## 步骤

### 1. 安装 istioctl

S3 工具桶不存在，改为操作机直接从 GitHub Releases 下载官方 release 包（同一个包里含 istioctl 二进制 + samples 目录下的 Bookinfo/addons manifest，后面步骤3、7直接复用）：

```bash
curl -sL https://github.com/istio/istio/releases/download/${ISTIO_VERSION}/istio-${ISTIO_VERSION}-linux-amd64.tar.gz -o /tmp/istio.tar.gz
tar -xzf /tmp/istio.tar.gz -C /tmp/
sudo cp /tmp/istio-${ISTIO_VERSION}/bin/istioctl /usr/local/bin/

istioctl version --remote=false
```

**预期输出**：打印 istioctl 客户端版本 `1.24.2`。

### 2. 安装 Istio 控制面

使用 `demo` profile（含完整组件）：

```bash
istioctl install \
  --set profile=demo \
  --set hub=${ISTIO_HUB} \
  -y

kubectl rollout status deployment/istiod -n istio-system --timeout=180s
kubectl get pods -n istio-system
```

**预期输出**：`deployment "istiod" successfully rolled out`；`istio-system` 命名空间下 `istiod`/`istio-ingressgateway`/`istio-egressgateway` 等 Pod 均 Running。

Ingress Gateway Service 改为内部访问（安全基线禁止 `0.0.0.0/0` 公网入站）：

```bash
kubectl -n istio-system annotate svc istio-ingressgateway \
  service.beta.kubernetes.io/aws-load-balancer-internal=true \
  service.beta.kubernetes.io/aws-load-balancer-scheme=internal \
  --overwrite

kubectl get svc istio-ingressgateway -n istio-system
```

**预期输出**：Service 标注成功；对应 NLB/ELB 的 scheme 变为 `internal`（可用 `aws elbv2 describe-load-balancers` 交叉确认）。

启用 namespace 自动注入 sidecar：

```bash
kubectl create namespace bookinfo
kubectl label namespace bookinfo istio-injection=enabled
```

**预期输出**：`namespace/bookinfo created`、`namespace/bookinfo labeled`。

### 3. 部署 Bookinfo 示例

Bookinfo 包含 4 个微服务（productpage / details / reviews / ratings），其中 reviews 有 3 个版本。Bookinfo manifest 用 release 包自带的 `samples/bookinfo/platform/kube/bookinfo.yaml`，把镜像引用替换成 ECR 中国（6个应用镜像需要预先转存，见前提条件）：

```bash
sed -E "s#docker.io/istio/examples-bookinfo-([a-z0-9-]+):1.20.2#${ECR_REGISTRY}/istio/examples-bookinfo-\1:1.20.2#g" \
  /tmp/istio-${ISTIO_VERSION}/samples/bookinfo/platform/kube/bookinfo.yaml > /tmp/bookinfo-cn.yaml

kubectl apply -f /tmp/bookinfo-cn.yaml -n bookinfo

kubectl get pods -n bookinfo -w
```

**预期输出**：6 个 Pod（productpage v1, details v1, reviews v1/v2/v3, ratings v1）全部 Running，每个 Pod `2/2` 容器（业务容器 + istio-proxy sidecar）。

创建 Gateway 暴露入口：

```bash
cat <<'EOF' | kubectl apply -f - -n bookinfo
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
EOF

INGRESS_HOST=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "内部访问地址（浏览器/公网场景）：http://${INGRESS_HOST}/productpage"

# 集群内验证一律用 ClusterIP，不用 LoadBalancer 主机名（原因见上方"中国区适配"两个实测发现）
# 且测试 Pod 必须建在没开 istio-injection 的命名空间（不能是 -n bookinfo）
INGRESS_CLUSTERIP=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.spec.clusterIP}')
kubectl run verify-tmp --rm -i --image=public.ecr.aws/docker/library/busybox:latest --restart=Never -- \
  wget -qO- --timeout=5 http://${INGRESS_CLUSTERIP}/productpage | head -c 200
```

**预期输出**：返回 productpage HTML 片段（含 `<title>Simple Bookstore App</title>` 等字样）。

> **两处已修正：** ① 测试 Pod 不能建在开了 `istio-injection=enabled` 的命名空间（如 `bookinfo`），否则自己被注入 sidecar 导致 `Connection refused`，应放到 `default`。② 集群内用 LoadBalancer 主机名测试会 DNS 解析失败，且 NLB 默认关闭跨可用区负载均衡；改用 ClusterIP 可同时绕开这两个问题，是集群内验证 Istio Ingress 更可靠的方式。

### 4. 流量管理：金丝雀发布

创建 DestinationRule 定义子集：

```bash
cat <<'EOF' | kubectl apply -f - -n bookinfo
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
EOF
```

**预期输出**：`destinationrule.networking.istio.io/reviews created`。

90% v1 + 10% v3：

```bash
cat <<'EOF' | kubectl apply -f - -n bookinfo
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v3
      weight: 10
EOF
```

**预期输出**：`virtualservice.networking.istio.io/reviews created`。

验证 v3 命中比例（从集群内发起）：

```bash
kubectl run verify-canary --rm -i --image=public.ecr.aws/docker/library/busybox:latest --restart=Never -- sh -c '
hit_v3=0
for i in $(seq 1 100); do
  v=$(wget -qO- http://'"${INGRESS_CLUSTERIP}"'/productpage | grep -oE "reviews-v[123]" | head -1)
  [ "$v" = "reviews-v3" ] && hit_v3=$((hit_v3+1))
done
echo "v3 命中: ${hit_v3} / 100（期望约 10）"
'
```

**预期输出**：`v3 命中: <约10> / 100`（8-12 之间均属正常波动范围）。

切到 50%/50%：

```bash
kubectl patch virtualservice reviews -n bookinfo --type=merge -p '
spec:
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v3
      weight: 50
'
```

**预期输出**：`virtualservice.networking.istio.io/reviews patched`。

全量切到 v3：

```bash
kubectl patch virtualservice reviews -n bookinfo --type=merge -p '
spec:
  http:
  - route:
    - destination:
        host: reviews
        subset: v3
      weight: 100
'
```

**预期输出**：`virtualservice.networking.istio.io/reviews patched`；重新执行验证脚本，v3 命中比例应接近 100/100。

### 5. mTLS 自动加密

启用 STRICT 模式：

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: bookinfo
spec:
  mtls:
    mode: STRICT
EOF
```

**预期输出**：`peerauthentication.security.istio.io/default created`。

验证未注入 sidecar 的 Pod 无法访问：

```bash
kubectl run test-no-sidecar --image=public.ecr.aws/docker/library/busybox:latest \
  -n default --restart=Never -- sleep 3600

kubectl wait --for=condition=Ready pod/test-no-sidecar -n default --timeout=60s

kubectl exec -n default test-no-sidecar -- \
  wget -qO- --timeout=5 http://productpage.bookinfo.svc.cluster.local:9080/productpage

kubectl delete pod test-no-sidecar -n default
```

**预期输出**：`wget` 连接失败（实测返回 `wget: error getting response: Connection reset by peer`，`exit code 1`；具体报错文案取决于 Envoy 版本，`can't connect`/超时/`Connection reset by peer` 都属于 STRICT mTLS 拒绝明文连接的正常表现），证明 STRICT mTLS 拒绝了非 mesh 内 Pod 的明文请求。

### 6. 服务间授权（AuthorizationPolicy）

```bash
cat <<'EOF' | kubectl apply -f - -n bookinfo
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: reviews-policy
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/bookinfo/sa/bookinfo-productpage"]
EOF
```

**预期输出**：`authorizationpolicy.security.istio.io/reviews-policy created`；后续只有 `productpage` 的 ServiceAccount 身份可以访问 `reviews` 服务，其余来源会被拒绝。

### 7. 可观测性：Kiali + Jaeger

Addon manifest 用 release 包自带的 `samples/addons/`（不需要额外 S3 tarball）。manifest 里的镜像 tag 与 ECR 中国已预置的版本不完全一致（`jaeger` manifest 默认引用 `1.58`，ECR 是 `1.76.0`；`kiali` manifest 默认引用 `v2.0`，ECR 是 `v1.89`；`prometheus` 主镜像 manifest 默认引用 `v2.54.1`，ECR 是 `v3.9.1`），直接 `sed` 替换成 ECR 已转存的 registry + tag，避免重复转存：

```bash
ADDONS_DIR=/tmp/istio-${ISTIO_VERSION}/samples/addons

sed "s#docker.io/jaegertracing/all-in-one:1.58#${ECR_REGISTRY}/jaegertracing/all-in-one:1.76.0#" \
  ${ADDONS_DIR}/jaeger.yaml > /tmp/jaeger-cn.yaml
sed "s#quay.io/kiali/kiali:v2.0#${ECR_REGISTRY}/kiali/kiali:v1.89#" \
  ${ADDONS_DIR}/kiali.yaml > /tmp/kiali-cn.yaml
sed -e "s#ghcr.io/prometheus-operator/prometheus-config-reloader:v0.76.0#${ECR_REGISTRY}/prometheus-operator/prometheus-config-reloader:v0.76.0#" \
    -e "s#prom/prometheus:v2.54.1#${ECR_REGISTRY}/prom/prometheus:v3.9.1#" \
    ${ADDONS_DIR}/prometheus.yaml > /tmp/prometheus-cn.yaml

kubectl apply -f /tmp/prometheus-cn.yaml
kubectl apply -f /tmp/jaeger-cn.yaml
kubectl apply -f /tmp/kiali-cn.yaml

kubectl rollout status deployment/kiali -n istio-system --timeout=120s
kubectl rollout status deployment/jaeger -n istio-system --timeout=120s
```

**预期输出**：`deployment "kiali" successfully rolled out`、`deployment "jaeger" successfully rolled out`。

端口转发访问（仅本地/操作机，不对外暴露）：

```bash
kubectl port-forward -n istio-system svc/kiali 20001:20001 &
echo "Kiali: http://localhost:20001/kiali"

kubectl port-forward -n istio-system svc/tracing 16686:80 &
echo "Jaeger: http://localhost:16686"
```

**预期输出**：Kiali Graph 展示 bookinfo 完整服务调用拓扑，含 mTLS 锁图标和实时 RPS；Jaeger 查询 productpage 的 trace，看到 productpage → details + reviews → ratings 的链路。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 所有核心资源创建成功且状态正常
- [ ] 验证检查点中的所有命令返回预期结果
- [ ] 理解 STRICT mTLS 如何拒绝非 mesh 内的明文流量

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get pods -n istio-system --no-headers \| grep Running \| wc -l \| tr -d ' '` | 大于等于 `1`（istiod + gateway + 可观测组件） |
| 2 | `kubectl get pods -n bookinfo --no-headers \| grep Running \| wc -l \| tr -d ' '` | `6` |
| 3 | `kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.metadata.annotations.service\.beta\.kubernetes\.io/aws-load-balancer-scheme}'` | `internal` |
| 4 | `kubectl get peerauthentication default -n bookinfo -o jsonpath='{.spec.mtls.mode}'` | `STRICT` |

---

## 实验总结

本 Lab 验证了 Istio 在中国区私有镜像仓库下的服务网格核心能力：流量管理（金丝雀）、mTLS 强制加密、服务间授权、可观测性，全程在内部 Ingress Gateway（无公网暴露）下完成验证。实测中确认了两个环境特性——Auto Mode 集群 VPC 的默认 DNS 解析器不解析内部 NLB 自动生成的主机名、NLB 默认关闭跨可用区负载均衡——并给出了绕开两者的可靠验证方式（用 ClusterIP）。作为可选 Lab，本 Lab 是本 workshop 的最后一个。

---

## 清理

```bash
kubectl delete -f /tmp/bookinfo.yaml -n bookinfo
kubectl delete namespace bookinfo
istioctl uninstall --purge -y
kubectl delete namespace istio-system
```

**预期输出**：`bookinfo`/`istio-system` 命名空间及全部 Istio 资源清除确认无残留。

> 确认 Ingress Gateway 自动创建的 NLB/ELB 已随 Service 删除释放。
