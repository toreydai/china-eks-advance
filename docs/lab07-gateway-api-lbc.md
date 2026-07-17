# Lab07 — Gateway API 和 AWS Load Balancer Controller 入口进阶

#### 更新时间: 2026-07-16
#### 基于EKS版本: EKS 1.35

## 实验简介

Gateway API 是 Ingress 的标准化替代方案，本 Lab 结合 AWS Load Balancer Controller 创建 ALB。

> **关键注意事项**：
> - LBC v3.3.0 的 Gateway API 支持（ALBGatewayAPI feature gate）通过 Helm 值 `--set controllerConfig.featureGates.ALBGatewayAPI=true` 启用，需要同时安装 Gateway API standard CRD（v1）。
> - GatewayClass 的 `controllerName` 必须为 **`gateway.k8s.aws/alb`**（不是 `elbv2.k8s.aws/alb`，那是 IngressClass 用的；也不是 `eks.amazonaws.com/alb`）。写错会导致 GatewayClass reconcile 静默跳过，`lastTransitionTime` 停留在 epoch 0，没有报错提示。
> - LBC v3.x 的 Gateway API 使用新模型：GatewayClass 需要通过 `parametersRef` 引用 `gateway.k8s.aws/v1beta1 LoadBalancerConfiguration` 对象。
> - **后端 Service 必须为 NodePort 类型**：LBC Gateway API 默认使用 Instance target type，要求 Service 有 NodePort（ClusterIP 会导致 `TargetGroup port is empty` 错误）。
> - LBC Helm 升级时若指定 `serviceAccount.create=false`，需提前手动创建 ServiceAccount `aws-load-balancer-controller`（该 SA 不会由 helm 管理）。
> - **安全适配**：`LoadBalancerConfiguration` 的 `scheme` 用 `internal`（中国区安全基线禁止 `0.0.0.0/0` 公网入站），验证从集群内部或 SSM/端口转发发起，不用公网 curl。

**实验目标：**
- 掌握 LBC v3.x Gateway API 支持的关键配置（controllerName、LoadBalancerConfiguration）
- 理解 Gateway API 与 Ingress 在 API 模型上的差异，及常见 Route 类型的适用场景
- 能够独立完成从安装 CRD 到验证内部 ALB 流量路由的完整闭环

**实验流程：**
1. 安装 Gateway API CRD
2. 部署测试服务
3. 升级 LBC 启用 ALBGatewayAPI feature gate
4. 创建 LoadBalancerConfiguration、GatewayClass、Gateway 和 HTTPRoute
5. 常见 Route 类型

**预计 AI 执行时长：** 15-20 分钟

## 前提条件

- 已安装 AWS Load Balancer Controller（Quickstart Demo02）
- 已下载 `~/gateway-api-standard-install.yaml`（Quickstart Demo01 §3 已预置）
- LBC 镜像版本 ≥ v3.3.0

```bash
export AWS_PROFILE=cn
export AWS_REGION=cn-northwest-1
export AWS_DEFAULT_REGION=cn-northwest-1
export CLUSTER_NAME=demo
export GW_NS=gateway-demo
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REGISTRY=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn
```

---

## 步骤

### 1. 安装 Gateway API CRD

```bash
kubectl -n kube-system get deployment aws-load-balancer-controller

kubectl apply -f ~/gateway-api-standard-install.yaml
kubectl get crd gateways.gateway.networking.k8s.io httproutes.gateway.networking.k8s.io
```

**预期输出**：`aws-load-balancer-controller` Deployment 已存在；两个 CRD（`gateways`/`httproutes`）均在 `kubectl get crd` 列表中。

### 2. 部署测试服务

```bash
kubectl create namespace ${GW_NS}
kubectl -n ${GW_NS} create deployment nginx --image=public.ecr.aws/docker/library/nginx:1.27-alpine
# 必须用 NodePort——LBC Gateway API 默认 Instance target type，ClusterIP 会报 TargetGroup port is empty
kubectl -n ${GW_NS} expose deployment nginx --port 80 --type NodePort
kubectl -n ${GW_NS} rollout status deployment/nginx
```

**预期输出**：`deployment "nginx" successfully rolled out`；`kubectl -n ${GW_NS} get svc nginx` 显示 `TYPE=NodePort`。

### 3. 升级 LBC 启用 ALBGatewayAPI feature gate

```bash
# 注意：serviceAccount.create=false 时需手动预创建 SA
kubectl get sa aws-load-balancer-controller -n kube-system 2>/dev/null || \
  kubectl create sa aws-load-balancer-controller -n kube-system

helm upgrade aws-load-balancer-controller /tmp/aws-load-balancer-controller.tgz \
  -n kube-system \
  --set clusterName=${CLUSTER_NAME} \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set image.repository=${ECR_REGISTRY}/aws-load-balancer-controller \
  --set image.tag=v3.3.0 \
  --set enableShield=false \
  --set enableWaf=false \
  --set enableWafv2=false \
  --set controllerConfig.featureGates.ALBGatewayAPI=true \
  --set replicaCount=1

kubectl rollout status deployment/aws-load-balancer-controller -n kube-system --timeout=120s
```

**预期输出**：`deployment "aws-load-balancer-controller" successfully rolled out`。

### 4. 创建 LoadBalancerConfiguration、GatewayClass、Gateway 和 HTTPRoute

```bash
# LoadBalancerConfiguration（LBC v3.x 必需，scheme 用 internal）
cat <<'YAML_EOF' | kubectl apply -f -
apiVersion: gateway.k8s.aws/v1beta1
kind: LoadBalancerConfiguration
metadata:
  name: alb-config
  namespace: gateway-demo
spec:
  scheme: internal
  ipAddressType: ipv4
YAML_EOF

# GatewayClass，controllerName 必须为 gateway.k8s.aws/alb
cat <<'YAML_EOF' | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: alb
spec:
  controllerName: gateway.k8s.aws/alb
  parametersRef:
    group: gateway.k8s.aws
    kind: LoadBalancerConfiguration
    name: alb-config
    namespace: gateway-demo
YAML_EOF

kubectl get gatewayclass alb
```

**预期输出**：`kubectl get gatewayclass alb` 的 `ACCEPTED` 列为 `True`（controllerName 正确时 LBC 立即 accept）。

```bash
cat <<'YAML_EOF' | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: alb-gateway
  namespace: gateway-demo
spec:
  gatewayClassName: alb
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nginx
  namespace: gateway-demo
spec:
  parentRefs:
  - name: alb-gateway
  rules:
  - backendRefs:
    - name: nginx
      port: 80
YAML_EOF
```

**预期输出**：`gateway.gateway.networking.k8s.io/alb-gateway created`、`httproute.gateway.networking.k8s.io/nginx created`；约 3-5 分钟后 `kubectl get gateway alb-gateway -n gateway-demo` 的 `PROGRAMMED` 列变为 `True`。

### 5. 常见 Route 类型

Gateway API 不只有 `HTTPRoute`。常见 Route 还包括：

- `GRPCRoute`：用于 gRPC 服务的 host/header/method 路由。
- `TLSRoute`：用于 TLS passthrough 场景，按 SNI 路由 TLS 流量。
- `TCPRoute` / `UDPRoute`：用于四层 TCP/UDP 流量，通常由支持四层负载均衡的 controller 实现。

本实验的 AWS Load Balancer Controller + ALB 路径重点验证 `HTTPRoute`。下面两个对象用于认识 API 形态；是否能被当前 controller 完整 reconcile，取决于 controller 对对应 Route kind 的支持情况。

**GRPCRoute 示例（可选）**：

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: grpc-example
  namespace: ${GW_NS}
spec:
  parentRefs:
  - name: alb-gateway
  hostnames:
  - grpc.example.com
  rules:
  - backendRefs:
    - name: nginx
      port: 80
EOF
```

**预期输出**：资源 apply 成功（`grpcroute.gateway.networking.k8s.io/grpc-example created`）；是否被 LBC 完整 reconcile 取决于当前版本对 GRPCRoute 的支持程度，仅用于认识 API 形态。

**TLSRoute 示例（可选）**：

```bash
cat <<EOF > /tmp/tlsroute-example.yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TLSRoute
metadata:
  name: tls-example
  namespace: ${GW_NS}
spec:
  parentRefs:
  - name: alb-gateway
  hostnames:
  - tls.example.com
  rules:
  - backendRefs:
    - name: nginx
      port: 80
EOF

kubectl get crd tlsroutes.gateway.networking.k8s.io \
  && kubectl apply -f /tmp/tlsroute-example.yaml \
  || echo "当前 CRD 未安装 TLSRoute，跳过可选示例"
```

**预期输出**：若集群已安装 TLSRoute CRD，资源创建成功；否则打印"当前 CRD 未安装 TLSRoute，跳过可选示例"，不算失败。

查看 Route 状态：

```bash
kubectl -n ${GW_NS} get httproute
kubectl -n ${GW_NS} get grpcroute || true
kubectl -n ${GW_NS} get tlsroute || true
kubectl -n ${GW_NS} describe httproute nginx
```

**预期输出**：`httproute` 列表中 `nginx` 状态正常，`describe` 输出的 `Status.Parents` 中 Gateway 已接受该 Route。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 所有核心资源创建成功且状态正常
- [ ] 验证检查点中的所有命令返回预期结果
- [ ] 理解 `controllerName` 配置错误会导致的静默失败现象

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get gatewayclass alb -o jsonpath='{.status.conditions[?(@.type=="Accepted")].status}'` | `True` |
| 2 | `kubectl get gateway alb-gateway -n ${GW_NS} -o jsonpath='{.status.conditions[?(@.type=="Programmed")].status}'` | `True` |
| 3 | `kubectl get gateway alb-gateway -n ${GW_NS} -o jsonpath='{.status.addresses[0].value}' \| grep -c .` | `1`（非空） |
| 4 | 见下方集群内 HTTP 验证 | `HTTP/1.1 200` |

```bash
kubectl -n ${GW_NS} get gateway alb-gateway
kubectl -n ${GW_NS} get httproute nginx

# 等待 ALB 就绪（3–5 分钟）
until ALB=$(kubectl -n ${GW_NS} get gateway alb-gateway \
  -o jsonpath='{.status.addresses[0].value}') && [ -n "$ALB" ]; do
  sleep 15
done

# 内部 ALB，从集群内验证（不用公网 curl）
kubectl run verify-tmp --rm -it --image=public.ecr.aws/docker/library/busybox:latest --restart=Never -- \
  wget -qO- --timeout=5 http://${ALB}
```

**预期输出**：返回 nginx 默认首页 HTML。

---

## 实验总结

本 Lab 验证了 LBC v3.x Gateway API 支持在中国区的完整链路：`controllerName` 正确配置、`LoadBalancerConfiguration` 新 CRD 模型、NodePort 后端要求，并在内部 scheme 下完成流量验证，不依赖公网暴露。Lab08 将学习 CodePipeline 进行 EKS CI/CD。

---

## 清理

```bash
kubectl delete namespace ${GW_NS}
kubectl delete -f ~/gateway-api-standard-install.yaml
```

**预期输出**：`gateway-demo` 命名空间及其下全部资源（Gateway/HTTPRoute/ALB）清除确认无残留。

> 实验结束后将 LBC 恢复标准配置（移除 `ALBGatewayAPI` feature gate），避免影响后续 Lab。
