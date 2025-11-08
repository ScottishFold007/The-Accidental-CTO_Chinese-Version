## 第22章:看门人——当API成为产品

### Part 1:增长的烦恼

小店通的成功带来了一个新问题——一个好问题,但仍然是个问题。

那是2024年初的一个周三下午,我正在和产品团队开会。产品负责人刘明兴奋地说:

"陈浩,好消息!我们签了三个大客户,他们都想通过API集成小店通到自己的系统。第一个是**快递通**,他们想自动同步订单到他们的系统。第二个是**易付通**,他们想成为我们的支付选项。第三个是**数析云**,他们想抓取我们的销售数据做分析。"

我笑了笑,但心里却涌起一阵担忧。"这确实是好消息。但我们的API准备好对外开放了吗?"

刘明困惑地看着我。"准备好?我们不是已经有API了吗?前端就在用啊。"

"是的,"我说,"但那些是**内部API**,为我们自己的前端设计的。对外开放API是完全不同的游戏。"

我走到白板前,开始列出挑战:

#### **对外API的挑战清单**

**1. 认证和授权**
```
问题:如何确保只有授权的客户能访问API?
现状:前端用JWT,但每个外部客户都需要不同的密钥
需求:API Key管理、权限控制、密钥轮换
```

**2. 限流(Rate Limiting)**
```
问题:如何防止某个客户占用所有资源?
现状:无限流,一个客户可以每秒发送10000个请求
需求:按客户限流(如每分钟1000次)、配额管理
```

**3. 计费**
```
问题:如何统计每个客户的API调用次数?
现状:无统计,不知道谁调用了多少次
需求:按调用次数计费、月度报表
```

**4. API版本管理**
```
问题:如何在不破坏现有客户的情况下升级API?
现状:所有客户用同一个API版本,改一个影响所有人
需求:v1、v2并存、废弃通知
```

**5. 文档和调试**
```
问题:客户如何知道怎么调用我们的API?
现状:内部API没有文档,靠口口相传
需求:自动生成文档、测试沙盒、错误码说明
```

**6. 监控和SLA**
```
问题:如何保证99.9%的API可用性?
现状:API挂了可能好几个小时才发现
需求:实时监控、告警、SLA承诺
```

**7. 协议转换**
```
问题:有些客户想用gRPC,有些想用REST
现状:只支持REST,要支持gRPC得重写代码
需求:协议网关、自动转换
```

看着这个清单,刘明的兴奋逐渐变成了担忧。"这...听起来工作量很大。我们需要多久能实现这些?"

"如果我们自己从头实现,"我说,"至少需要三个月,两个全职工程师。但还有更好的办法。"

我在白板上写下了两个字:

**API网关 (API Gateway)**

### Part 2:API网关——不只是路由

"API网关,"我解释道,"是所有API请求的**单一入口**。它像一个聪明的前台接待,不仅指引客户去正确的服务,还负责安全检查、计数、限流等所有通用功能。"

#### **Nginx vs. API网关**

刘明问:"我们不是已经用Nginx做负载均衡了吗?这不够吗?"

"好问题,"我说。"让我们对比一下:"

```mermaid
%%{init: {'theme':'dark'}}%%
graph TB
    subgraph Nginx_Role[Nginx的职责]
        N1[HTTP路由]
        N2[负载均衡]
        N3[SSL终止]
        N4[静态文件服务]
        N5[反向代理]
    end
    
    subgraph Gateway_Role[API网关的职责]
        G1[HTTP路由 ✅]
        G2[负载均衡 ✅]
        G3[SSL终止 ✅]
        G4[===以下Nginx需要Lua脚本===]
        G5[身份认证 JWT/OAuth/API Key]
        G6[限流 per-consumer/per-endpoint]
        G7[熔断 Circuit Breaker]
        G8[请求/响应转换]
        G9[协议转换 REST↔gRPC]
        G10[API版本管理]
        G11[监控和分析]
        G12[开发者门户]
    end
    
    Nginx_Role -.->|功能有限| Limit[需要大量<br/>自定义Lua代码]
    Gateway_Role -.->|开箱即用| Easy[插件生态<br/>配置即可]
    
    style Nginx_Role fill:#374151,stroke:#f59e0b,color:#f3f4f6,stroke-width:2px
    style Gateway_Role fill:#1f2937,stroke:#10b981,color:#f3f4f6,stroke-width:3px
    style Limit fill:#1f2937,stroke:#ef4444,color:#f3f4f6,stroke-width:2px
    style Easy fill:#1f2937,stroke:#10b981,color:#f3f4f6,stroke-width:2px
```

**Nginx擅长的:**
- ✅ 简单高效的HTTP路由
- ✅ 静态文件服务
- ✅ 反向代理

**Nginx不擅长的:**
- ❌ 身份认证(需要手写Lua脚本)
- ❌ 精细化限流(只支持IP级别,不支持per-API-key)
- ❌ 请求/响应转换(需要复杂的Lua代码)
- ❌ 协议转换(REST→gRPC几乎不可能)
- ❌ API文档生成(完全不支持)

"所以,"我总结道,"Nginx是一个优秀的**反向代理**,但不是一个完整的**API管理平台**。对于对外API,我们需要后者。"

#### **API网关选型**

我在白板上列出了主流的API网关方案:

| 方案 | 类型 | 优势 | 劣势 | 适用场景 |
|------|------|------|------|---------|
| **Kong** | 开源 | 功能最全、插件丰富(1800+)、Kubernetes原生 | 需要PostgreSQL | 中大型企业 |
| **Apache APISIX** | 开源 | 性能最高、etcd存储、国内支持好 | 社区较小 | 性能要求高 |
| **Tyk** | 开源/商业 | Go编写、性能好 | 企业版功能少 | 中型企业 |
| **阿里云 API Gateway** | SaaS | 零运维、与阿里云集成好 | 贵、锁定阿里云 | 阿里云用户 |
| **Apigee** (Google) | SaaS | 功能强大、企业级 | 非常贵($$$) | 大型企业 |

"我推荐**Kong**,"我说。"它是CNCF的项目,被Netflix、Yahoo、NASA等大公司使用,插件生态最成熟,而且是真正的开源。"

团队同意了。我们决定引入Kong作为对外API的统一网关。

### Part 3:Kong的核心概念

在动手之前,我花了一个下午学习Kong的架构。它的概念模型很优雅:

#### **Kong的四层抽象**

```mermaid
%%{init: {'theme':'dark'}}%%
graph TB
    Client[外部客户端] --> Kong{Kong Gateway}
    
    Kong --> Route1[Route<br/>/api/v1/orders<br/>匹配规则]
    Kong --> Route2[Route<br/>/api/v1/products<br/>匹配规则]
    
    Route1 --> Service1[Service<br/>order-service<br/>上游地址]
    Route2 --> Service2[Service<br/>product-service<br/>上游地址]
    
    Service1 --> Upstream1[Upstream<br/>负载均衡池]
    Service2 --> Upstream2[Upstream<br/>负载均衡池]
    
    Upstream1 --> Backend1[Backend 1<br/>order-pod-1]
    Upstream1 --> Backend2[Backend 2<br/>order-pod-2]
    Upstream1 --> Backend3[Backend 3<br/>order-pod-3]
    
    Service1 -.->|应用插件| Plugins[Plugins<br/>认证、限流、<br/>监控等]
    
    Consumer[Consumer<br/>API消费者<br/>快递通、易付通]
    Consumer -.->|身份标识| Kong
    
    style Kong fill:#1f2937,stroke:#60a5fa,color:#f3f4f6,stroke-width:3px
    style Plugins fill:#374151,stroke:#10b981,color:#f3f4f6,stroke-width:2px
    style Consumer fill:#374151,stroke:#f59e0b,color:#f3f4f6,stroke-width:2px
```

**1. Service(服务)**

Service定义了**上游服务的位置**,比如:

```yaml
name: order-service
url: http://order-service.production.svc.cluster.local:8080
protocol: http
retries: 3
connect_timeout: 5000
read_timeout: 60000
write_timeout: 60000
```

**2. Route(路由)**

Route定义了**如何匹配客户端请求**到Service:

```yaml
name: orders-api-v1
service: order-service
paths:
  - /api/v1/orders
methods:
  - GET
  - POST
  - PUT
hosts:
  - api.xiaodiantong.com
```

请求`GET https://api.xiaodiantong.com/api/v1/orders`会匹配这个Route,转发到order-service。

**3. Consumer(消费者)**

Consumer代表**API的使用者**,可以是外部公司、合作伙伴、或者内部应用:

```yaml
name: kuaidi-logistics
custom_id: partner-001
tags:
  - partner
  - logistics
```

每个Consumer可以有自己的认证凭证(API Key、JWT、OAuth Token)和限流配额。

**4. Plugin(插件)**

Plugin提供了**额外的功能**,可以应用到Service、Route或Consumer:

```yaml
# 应用到Service级别(所有相关Route生效)
service: order-service
plugins:
  - name: jwt
  - name: rate-limiting
  - name: prometheus
```

Kong有1800+插件,覆盖几乎所有需求。

#### **Kong的工作流程**

当一个API请求到达Kong时,会经过以下步骤:

```
1. 接收请求
   ↓
2. 路由匹配 (找到匹配的Route)
   ↓
3. 认证检查 (JWT/API Key插件)
   ↓
4. 限流检查 (Rate Limiting插件)
   ↓
5. 请求转换 (Request Transformer插件)
   ↓
6. 转发到上游服务
   ↓
7. 接收上游响应
   ↓
8. 响应转换 (Response Transformer插件)
   ↓
9. 返回给客户端
   ↓
10. 记录日志和指标 (Logging/Prometheus插件)
```

每个步骤都可以通过插件定制。这种设计非常灵活。

### Part 4:部署Kong

Kong有两种部署模式:

1. **DB模式**:使用PostgreSQL存储配置,支持多Kong实例,推荐用于生产
2. **DB-less模式**:配置存储在YAML文件中,无需数据库,适合简单场景

我们选择DB模式,因为需要多实例高可用。

#### **部署PostgreSQL**

Kong需要PostgreSQL存储配置。我们创建一个专用的数据库:

```bash
# 在Kubernetes中部署PostgreSQL
kubectl create namespace gateway

# 创建PostgreSQL密码Secret
kubectl create secret generic kong-postgres \
  --from-literal=password=$(openssl rand -base64 32) \
  -n gateway

# 部署PostgreSQL
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: gateway
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_DB
          value: kong
        - name: POSTGRES_USER
          value: kong
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: kong-postgres
              key: password
        ports:
        - containerPort: 5432
          name: postgres
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
          subPath: postgres
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: gateway
spec:
  type: ClusterIP
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: postgres
EOF
```

#### **初始化Kong数据库**

```bash
# 运行Kong migrations(初始化数据库表)
kubectl run kong-migrations \
  --image=kong:3.5 \
  --restart=Never \
  --namespace=gateway \
  --env="KONG_DATABASE=postgres" \
  --env="KONG_PG_HOST=postgres.gateway.svc.cluster.local" \
  --env="KONG_PG_DATABASE=kong" \
  --env="KONG_PG_USER=kong" \
  --env="KONG_PG_PASSWORD=$(kubectl get secret kong-postgres -n gateway -o jsonpath='{.data.password}' | base64 -d)" \
  --command -- kong migrations bootstrap

# 等待migrations完成
kubectl wait --for=condition=complete job/kong-migrations -n gateway --timeout=120s

# 清理migrations Job
kubectl delete pod kong-migrations -n gateway
```

#### **部署Kong Gateway**

```yaml
# kong-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kong-env
  namespace: gateway
data:
  KONG_DATABASE: "postgres"
  KONG_PG_HOST: "postgres.gateway.svc.cluster.local"
  KONG_PG_DATABASE: "kong"
  KONG_PG_USER: "kong"
  KONG_PROXY_ACCESS_LOG: "/dev/stdout"
  KONG_ADMIN_ACCESS_LOG: "/dev/stdout"
  KONG_PROXY_ERROR_LOG: "/dev/stderr"
  KONG_ADMIN_ERROR_LOG: "/dev/stderr"
  KONG_ADMIN_LISTEN: "0.0.0.0:8001"
  KONG_PROXY_LISTEN: "0.0.0.0:8000, 0.0.0.0:8443 ssl"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kong
  namespace: gateway
  labels:
    app: kong
spec:
  replicas: 3  # 高可用部署
  selector:
    matchLabels:
      app: kong
  template:
    metadata:
      labels:
        app: kong
    spec:
      containers:
      - name: kong
        image: kong:3.5
        envFrom:
        - configMapRef:
            name: kong-env
        env:
        - name: KONG_PG_PASSWORD
          valueFrom:
            secretKeyRef:
              name: kong-postgres
              key: password
        ports:
        - name: proxy
          containerPort: 8000
          protocol: TCP
        - name: proxy-ssl
          containerPort: 8443
          protocol: TCP
        - name: admin
          containerPort: 8001
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /status
            port: 8001
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /status
            port: 8001
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
---
# Kong Proxy Service (对外暴露API)
apiVersion: v1
kind: Service
metadata:
  name: kong-proxy
  namespace: gateway
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  ports:
  - name: proxy
    port: 80
    targetPort: 8000
    protocol: TCP
  - name: proxy-ssl
    port: 443
    targetPort: 8443
    protocol: TCP
  selector:
    app: kong
---
# Kong Admin API Service (内部管理)
apiVersion: v1
kind: Service
metadata:
  name: kong-admin
  namespace: gateway
spec:
  type: ClusterIP
  ports:
  - name: admin
    port: 8001
    targetPort: 8001
    protocol: TCP
  selector:
    app: kong
```

**部署Kong:**

```bash
kubectl apply -f kong-deployment.yaml

# 等待Kong Pod就绪
kubectl wait --for=condition=ready pod -l app=kong -n gateway --timeout=180s

# 获取Kong Proxy的外部IP
kubectl get svc kong-proxy -n gateway

# 输出类似:
# NAME         TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)
# kong-proxy   LoadBalancer   10.0.123.45    52.66.123.45     80:31234/TCP,443:32456/TCP

# 测试Kong是否工作
curl -i http://52.66.123.45/
# 应该返回: {"message":"no Route matched with those values"}
# 这是正常的,因为我们还没配置任何Route
```

### Part 5:配置Kong——声明式管理

Kong提供了多种配置方式:
- Admin API:通过REST API配置
- **deck CLI**:声明式配置(推荐,类似Kubernetes YAML)
- Konga/Kong Manager:Web UI

我们选择**deck CLI**,因为它支持GitOps,配置可以版本控制。

#### **安装deck**

```bash
# 下载deck
curl -sL https://github.com/kong/deck/releases/download/v1.28.0/deck_1.28.0_linux_amd64.tar.gz | tar xz -C /tmp
sudo mv /tmp/deck /usr/local/bin/

# 验证安装
deck version
```

#### **导出当前配置(空配置)**

```bash
# 设置Kong Admin API地址
export KONG_ADMIN_URL="http://localhost:8001"

# 如果Admin API不在本地,需要port-forward
kubectl port-forward svc/kong-admin -n gateway 8001:8001 &

# 导出配置
deck dump --output-file kong.yaml
```

#### **创建第一个API配置**

让我们为小店通的Order API配置Kong:

```yaml
# kong.yaml
_format_version: "3.0"

# 定义上游服务
services:
  - name: order-service
    url: http://order-service.production.svc.cluster.local:8080
    protocol: http
    connect_timeout: 5000
    read_timeout: 60000
    write_timeout: 60000
    retries: 3
    
    # 定义路由规则
    routes:
      - name: orders-api-v1
        paths:
          - /api/v1/orders
        methods:
          - GET
          - POST
          - PUT
          - DELETE
        strip_path: false
        preserve_host: false
        protocols:
          - http
          - https
    
    # 应用插件
    plugins:
      # 1. JWT认证
      - name: jwt
        config:
          claims_to_verify:
            - exp
          key_claim_name: iss
          secret_is_base64: false
      
      # 2. 限流(所有用户共享)
      - name: rate-limiting
        config:
          minute: 1000          # 每分钟1000次
          policy: redis         # 使用Redis存储计数器(分布式)
          redis_host: redis.middleware.svc.cluster.local
          redis_port: 6379
          redis_database: 2
          fault_tolerant: true  # Redis故障时不阻断请求
      
      # 3. 请求大小限制
      - name: request-size-limiting
        config:
          allowed_payload_size: 10  # 最大10MB
      
      # 4. CORS支持
      - name: cors
        config:
          origins:
            - "https://xiaodiantong.com"
            - "https://*.xiaodiantong.com"
          methods:
            - GET
            - POST
            - PUT
            - DELETE
            - OPTIONS
          headers:
            - Authorization
            - Content-Type
            - X-Request-ID
          exposed_headers:
            - X-RateLimit-Remaining
            - X-RateLimit-Reset
          credentials: true
          max_age: 3600
      
      # 5. Prometheus监控
      - name: prometheus
        config:
          per_consumer: true  # 按Consumer统计
      
      # 6. 请求ID(追踪)
      - name: correlation-id
        config:
          header_name: X-Request-ID
          generator: uuid#counter
          echo_downstream: true
      
      # 7. 请求/响应转换
      - name: request-transformer
        config:
          add:
            headers:
              - "X-Service-Version:3.0"
              - "X-Forwarded-By:Kong"
      
      # 8. 响应缓存(可选)
      - name: proxy-cache
        config:
          strategy: memory
          content_type:
            - "application/json"
          cache_ttl: 300        # 缓存5分钟
          cache_control: true

# 定义API消费者
consumers:
  - username: kuaidi-logistics
    custom_id: partner-001
    tags:
      - partner
      - logistics
    
    # 为这个Consumer创建JWT凭证
    jwt_secrets:
      - key: kuaidi-jwt-key
        secret: "your-jwt-secret-here-replace-this"  # 生产环境用强密钥
        algorithm: HS256
    
    # 为这个Consumer单独配置限流
    plugins:
      - name: rate-limiting
        config:
          minute: 2000  # 快递通每分钟2000次(更高配额)
          policy: redis
          redis_host: redis.middleware.svc.cluster.local
          redis_port: 6379
  
  - username: yifutong-payment
    custom_id: partner-002
    tags:
      - partner
      - payment
    
    jwt_secrets:
      - key: yifutong-jwt-key
        secret: "another-strong-jwt-secret"
        algorithm: HS256
    
    plugins:
      - name: rate-limiting
        config:
          minute: 5000  # 易付通每分钟5000次(支付服务需要更高配额)

# 定义Upstream(负载均衡池)
upstreams:
  - name: order-service-pool
    algorithm: round-robin  # 轮询算法
    slots: 10000
    
    # 健康检查
    healthchecks:
      active:
        type: http
        http_path: /health
        timeout: 1
        concurrency: 10
        healthy:
          interval: 5      # 每5秒检查一次
          successes: 2     # 2次成功标记为健康
        unhealthy:
          interval: 5
          http_failures: 3  # 3次失败标记为不健康
          timeouts: 3
      passive:
        type: http
        healthy:
          successes: 5
        unhealthy:
          http_failures: 5
          timeouts: 5
    
    # 目标服务器列表(Kubernetes自动发现,也可以手动指定)
    targets:
      - target: order-service.production.svc.cluster.local:8080
        weight: 100
```

**应用配置到Kong:**

```bash
# 验证配置文件语法
deck validate --kong-addr http://localhost:8001 kong.yaml

# 预览变更(不实际应用)
deck diff --kong-addr http://localhost:8001 kong.yaml

# 应用配置
deck sync --kong-addr http://localhost:8001 kong.yaml

# 输出:
# creating service order-service
# creating route orders-api-v1
# creating plugin jwt (global)
# creating plugin rate-limiting (global)
# creating consumer kuaidi-logistics
# ...
# Summary:
#   Created: 15
#   Updated: 0
#   Deleted: 0
```

配置立即生效!Kong现在会:
1. 接收`/api/v1/orders`的请求
2. 验证JWT Token
3. 检查限流配额
4. 转发到order-service
5. 记录Prometheus指标

### Part 6:为外部客户生成JWT Token

外部客户需要JWT Token来认证。我们创建一个简单的脚本生成Token:

```python
# generate_jwt.py
import jwt
import datetime
import sys

def generate_jwt_token(consumer_key, secret, expiry_days=365):
    """
    为Kong Consumer生成JWT Token
    
    Args:
        consumer_key: Consumer的JWT key(如kuaidi-jwt-key)
        secret: JWT secret
        expiry_days: Token有效期(天)
    """
    payload = {
        "iss": consumer_key,  # Issuer(key_claim_name)
        "exp": datetime.datetime.utcnow() + datetime.timedelta(days=expiry_days),
        "iat": datetime.datetime.utcnow(),
        "sub": "api-access",
    }
    
    token = jwt.encode(payload, secret, algorithm="HS256")
    return token

if __name__ == "__main__":
    if len(sys.argv) != 3:
        print("Usage: python generate_jwt.py <consumer_key> <secret>")
        sys.exit(1)
    
    consumer_key = sys.argv[1]
    secret = sys.argv[2]
    
    token = generate_jwt_token(consumer_key, secret)
    print(f"JWT Token for {consumer_key}:")
    print(token)
    print(f"\n使用方法:")
    print(f"curl -H 'Authorization: Bearer {token}' https://api.xiaodiantong.com/api/v1/orders")

# 使用示例:
# python generate_jwt.py kuaidi-jwt-key your-jwt-secret-here
```

**生成快递通的Token:**

```bash
python generate_jwt.py kuaidi-jwt-key your-jwt-secret-here

# 输出:
# JWT Token for kuaidi-jwt-key:
# eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzaGlwZmFzdC1qd3Qta2V5IiwiZXhwIjoxNzM1NjgwMDAwLCJpYXQiOjE3MDQxNDQwMDAsInN1YiI6ImFwaS1hY2Nlc3MifQ.XYZ...
```

把这个Token发给快递通团队,他们就可以调用API了:

```bash
curl -H "Authorization: Bearer eyJhbGciOi..." \
     https://api.xiaodiantong.com/api/v1/orders
```

### Part 7:高级功能——让Kong更强大

配置好基础API后,我们开始探索Kong的高级功能。

#### **功能1:熔断器(Circuit Breaker)**

当上游服务不稳定时,熔断器可以防止雪崩:

```yaml
plugins:
  - name: circuit-breaker
    service: order-service
    config:
      threshold: 50           # 错误率阈值(50%)
      window_size: 60         # 窗口大小(60秒)
      minimum_requests: 10    # 最小请求数(10次后才开始统计)
      break_time: 30          # 熔断时长(30秒)
      error_codes:            # 哪些HTTP状态码算"错误"
        - 500
        - 502
        - 503
        - 504
```

**工作原理:**

```
正常状态 → 当60秒内>10个请求且错误率>50% → 熔断状态
熔断状态 → 返回503给客户端,不调用上游 → 持续30秒
熔断状态 → 30秒后 → 半开状态(尝试一个请求)
半开状态 → 如果成功 → 恢复正常状态
半开状态 → 如果失败 → 重新熔断30秒
```

这保护了上游服务,避免在它已经过载时继续发送请求。

#### **功能2:API版本管理**

我们可以同时运行v1和v2 API:

```yaml
services:
  # v1 API(旧版,维护模式)
  - name: order-service-v1
    url: http://order-service-v1.production.svc:8080
    routes:
      - name: orders-v1
        paths:
          - /api/v1/orders
        plugins:
          - name: response-transformer
            config:
              add:
                headers:
                  - "X-API-Version:1.0"
                  - "X-Deprecation-Warning:v1 API will be deprecated on 2024-12-31"
  
  # v2 API(新版,推荐)
  - name: order-service-v2
    url: http://order-service-v2.production.svc:8080
    routes:
      - name: orders-v2
        paths:
          - /api/v2/orders
        plugins:
          - name: response-transformer
            config:
              add:
                headers:
                  - "X-API-Version:2.0"
```

客户端可以选择调用`/api/v1/orders`或`/api/v2/orders`,我们在响应头中提醒v1即将废弃。

#### **功能3:协议转换(REST → gRPC)**

某些客户想用高性能的gRPC协议。Kong可以接收REST请求,转换为gRPC调用上游:

```yaml
services:
  - name: order-service-grpc
    url: grpc://order-service.production.svc:9090
    protocol: grpc
    
    routes:
      - name: orders-grpc
        paths:
          - /api/v1/orders
        protocols:
          - http
          - https
    
    plugins:
      - name: grpc-gateway
        config:
          proto: "/path/to/order.proto"  # gRPC定义文件
```

客户端仍然发送REST请求,Kong自动转换为gRPC。

#### **功能4:响应缓存**

对于不常变化的数据,可以在Kong层缓存:

```yaml
plugins:
  - name: proxy-cache
    route: product-list
    config:
      strategy: redis
      redis:
        host: redis.middleware.svc
        port: 6379
        database: 3
      content_type:
        - "application/json"
      cache_ttl: 600        # 缓存10分钟
      cache_control: true   # 遵守Cache-Control header
      request_method:
        - GET
        - HEAD
      response_code:
        - 200
        - 301
        - 302
```

第一个请求会转发到上游,后续请求直接从Redis返回,大幅降低上游负载。

#### **功能5:自定义插件(Lua)**

如果现有插件不满足需求,可以用Lua编写自定义插件。

**示例:商户配额管理插件**

```lua
-- kong/plugins/merchant-quota/handler.lua
local BasePlugin = require "kong.plugins.base_plugin"
local redis = require "resty.redis"

local MerchantQuotaHandler = BasePlugin:extend()

MerchantQuotaHandler.PRIORITY = 1000
MerchantQuotaHandler.VERSION = "1.0.0"

function MerchantQuotaHandler:access(conf)
  MerchantQuotaHandler.super.access(self)
  
  -- 从JWT中提取商户ID
  local merchant_id = kong.request.get_header("X-Merchant-ID")
  if not merchant_id then
    return kong.response.exit(400, {message = "Missing X-Merchant-ID header"})
  end
  
  -- 连接Redis
  local red = redis:new()
  red:set_timeout(1000)
  local ok, err = red:connect(conf.redis_host, conf.redis_port)
  
  if not ok then
    kong.log.err("Failed to connect to Redis: ", err)
    -- fault_tolerant模式:Redis故障时放行请求
    if conf.fault_tolerant then
      return
    end
    return kong.response.exit(500, {message = "Internal error"})
  end
  
  -- 检查月度配额
  local quota_key = "merchant:quota:" .. merchant_id .. ":" .. os.date("%Y-%m")
  local remaining = red:get(quota_key)
  
  if remaining == ngx.null then
    -- 初始化配额(从数据库读取商户套餐)
    local quota_limit = get_merchant_quota_limit(merchant_id)  -- 假设10000
    red:set(quota_key, quota_limit)
    red:expireat(quota_key, get_month_end_timestamp())
    remaining = quota_limit
  else
    remaining = tonumber(remaining)
  end
  
  -- 检查配额
  if remaining <= 0 then
    kong.log.warn("Merchant ", merchant_id, " exceeded quota")
    return kong.response.exit(429, {
      message = "Monthly quota exceeded",
      quota_limit = get_merchant_quota_limit(merchant_id),
      quota_remaining = 0,
      quota_reset = red:ttl(quota_key)
    })
  end
  
  -- 扣减配额
  red:decr(quota_key)
  
  -- 设置响应头
  kong.response.set_header("X-Quota-Remaining", remaining - 1)
  kong.response.set_header("X-Quota-Reset", red:ttl(quota_key))
  
  red:set_keepalive(10000, 100)
end

return MerchantQuotaHandler
```

**安装自定义插件:**

```bash
# 将插件代码打包到Kong镜像
# Dockerfile
FROM kong:3.5
COPY kong/plugins/merchant-quota /usr/local/share/lua/5.1/kong/plugins/merchant-quota

# 重新构建Kong镜像
docker build -t my-kong:3.5 .

# 在Kong配置中启用插件
# kong.conf
plugins = bundled,merchant-quota
```

### Part 8:监控Kong

Kong本身也需要监控。我们使用Prometheus插件导出指标:

```yaml
# 在全局级别启用Prometheus
_format_version: "3.0"

plugins:
  - name: prometheus
    config:
      per_consumer: true  # 按Consumer统计
      status_code_metrics: true
      latency_metrics: true
      bandwidth_metrics: true
```

**Kong导出的关键指标:**

```promql
# 请求速率(按Service)
sum(rate(kong_http_requests_total[5m])) by (service)

# 错误率
sum(rate(kong_http_requests_total{code=~"5.."}[5m])) 
/ 
sum(rate(kong_http_requests_total[5m]))

# P95延迟
histogram_quantile(0.95, sum(rate(kong_http_request_duration_ms_bucket[5m])) by (le, service))

# 限流拒绝数
rate(kong_rate_limiting_exceeded_total[5m])

# 按Consumer的请求数
sum(rate(kong_http_requests_total[5m])) by (consumer)
```

**Grafana仪表盘(Kong官方模板):**

```bash
# 导入Kong官方仪表盘
# Grafana ID: 7424 (Kong Official Dashboard)
```

仪表盘显示:
- 总请求数、成功率、错误率
- 每个Service的QPS和延迟
- 每个Consumer的使用情况
- 限流统计
- 上游服务健康状态

### Part 9:真实案例——快递通集成

两周后,快递通的工程师张伟联系我们,准备集成API。

**需求:**
1. 自动同步订单到快递通系统
2. 每分钟最多2000次API调用
3. 需要webhook通知新订单

**第1步:创建Consumer并生成JWT Token**

我已经在Kong配置中定义了快递通的Consumer。我运行脚本生成Token:

```bash
python generate_jwt.py kuaidi-jwt-key <secret>

# 将生成的Token发送给张伟
```

**第2步:张伟测试API**

```bash
# 张伟的测试脚本
curl -H "Authorization: Bearer eyJhbGci..." \
     https://api.xiaodiantong.com/api/v1/orders?status=pending

# 响应:
{
  "orders": [
    {"id": "ORD-12345", "status": "pending", "total": 1500},
    ...
  ],
  "total_count": 45
}

# 响应头包含配额信息:
X-RateLimit-Remaining: 1999
X-RateLimit-Reset: 45
X-Quota-Remaining: 9987
```

**第3步:配置Webhook**

张伟需要实时通知。我们为快递通配置webhook插件:

```yaml
consumers:
  - username: kuaidi-logistics
    plugins:
      - name: webhook
        config:
          url: https://webhooks.kuaidi.com/xiaodiantong/orders
          method: POST
          headers:
            - "Authorization: Bearer kuaidi-webhook-secret"
          events:
            - order.created
            - order.updated
          retry_count: 3
          timeout: 5000
```

现在,每当有新订单创建,Kong会自动POST到快递通的webhook端点。

**第4步:监控集成**

在Grafana中,我创建了一个快递通专属面板:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
快递通 API集成监控
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
今日调用: 45,234次
配额使用: 78.4% (9,832 / 12,544)
错误率: 0.02% (9次5xx错误)
P95延迟: 127ms
限流拒绝: 3次

最近错误:
- 2024-01-15 14:23:12 | 503 Service Unavailable | order-service超时
- 2024-01-15 11:45:33 | 500 Internal Server Error | 数据库连接失败
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

几天后,张伟发来感谢邮件:

> "集成非常顺利!API文档清晰,Token认证简单,限流和配额信息让我们能够合理规划调用频率。webhook实时性很好,延迟在1秒以内。非常感谢小店通团队的支持!"

这就是API网关的价值——**让集成变得简单**。

### Part 10:真实收益——从混乱到秩序

六个月后,我们已经接入了15个外部合作伙伴。我做了一次回顾分析:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Kong API网关收益分析
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
指标                          引入前         引入后        改善
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
新客户集成时间                3-4周         1-2天         93% ↓
认证系统开发时间              8人天         0人天         100% ↓
限流功能开发时间              5人天         0人天         100% ↓
API文档维护时间/周           8小时         1小时         87.5% ↓
API调用统计准确性             ~70%          99.9%         42.7% ↑
安全事件数/月                 3次           0次           100% ↓

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
新增能力(以前无法实现)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ 按消费者的精细化限流和配额
✅ JWT/OAuth认证(以前只支持API Key)
✅ 熔断保护(防止上游服务雪崩)
✅ API版本管理(v1、v2并存)
✅ 实时API调用监控和告警
✅ 自动生成API文档

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
成本
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
基础设施成本/月              $0            $180          新增
(Kong服务器 + PostgreSQL)                  (3×$60)

工程师工时节省/月            0              48小时        节省
(不需要重复开发认证/限流)                  ($3,840)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ROI计算:
  成本: $180/月
  节省: $3,840/月
  净收益: $3,660/月 = $43,920/年
  投资回报率: 2,033%
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

但最重要的收益是**业务增长**。有了稳定、专业的API,我们吸引了更多合作伙伴:

- 快递通(物流):每天同步5000+订单
- 易付通(支付):处理30%的支付交易
- 数析云(分析):为2000+商户提供数据洞察
- 另外12个合作伙伴:涵盖CRM、营销、库存管理等

**API成为了一条新的收入渠道**。我们开始向高级合作伙伴收取API使用费:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
小店通 API定价(2024年)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
套餐         月度配额      价格/月       超额费用
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Free         10,000次      $0           不支持
Startup      100,000次     $99          $0.002/次
Business     500,000次     $399         $0.0015/次
Enterprise   自定义        面议         自定义
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

当月API收入: $127,000
年化收入: $1,524,000
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

API从成本中心变成了利润中心。

---

<div style="border: 2px solid #f59e0b; border-radius: 8px; padding: 20px; margin: 30px 0; background: linear-gradient(to right, #78350f08, #92400e08);">

### 📌 编者注:API网关实战完整指南

*本指南提供了Kong API Gateway的完整部署和配置经验,帮助你在生产环境中快速搭建企业级API管理平台。*

---

#### **一、快速决策:我需要API网关吗?**

| 场景 | 是否需要 | 替代方案 |
|------|---------|---------|
| **<5个内部API** | ❌ 不需要 | Nginx反向代理足够 |
| **5-20个微服务API** | ⚠️ 可选 | 看是否有对外API需求 |
| **对外开放API** | ✅ 强烈推荐 | 无好的替代方案 |
| **多个API消费者** | ✅ 需要 | 需要区分消费者身份和配额 |
| **需要精细化限流** | ✅ 需要 | Nginx的限流太简单 |
| **API计费需求** | ✅ 需要 | 需要统计每个客户的调用量 |

**小店通的场景:**8个微服务 + 对外开放API + 15个外部客户 → 必须使用API网关

---

#### **二、API网关选型对比**

| 特性 | Kong | Apache APISIX | Tyk | 阿里云 API Gateway |
|------|------|---------------|-----|-----------------|
| **开源** | ✅ Apache 2.0 | ✅ Apache 2.0 | ✅/❌ MPL/商业 | ❌ 专有 |
| **语言** | Lua (OpenResty) | Lua (OpenResty) | Go | N/A (SaaS) |
| **配置存储** | PostgreSQL/Cassandra | etcd | Redis | DynamoDB |
| **性能** | 高 (10K+ RPS) | 极高 (20K+ RPS) | 高 (15K+ RPS) | 中 |
| **插件数量** | 1800+ | 80+ | 50+ | 100+ |
| **学习曲线** | 中等 | 中等 | 低 | 低 |
| **K8s集成** | ✅ Ingress Controller | ✅ 原生支持 | ✅ 原生支持 | ❌ |
| **社区活跃度** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **企业支持** | ✅ Kong Inc. | ✅ 支流科技 | ✅ Tyk | ✅ 阿里云 |
| **成本(自托管)** | 低 | 低 | 低 | N/A |
| **成本(托管)** | $$$  | $$ | $$ | $$ |

**小店通的选择:**Kong(插件生态最丰富、社区最活跃、Kubernetes集成好)

---

#### **三、Kong部署Checklist**

##### **阶段1:环境准备(半天)**

- [ ] **选择部署模式**
  - DB模式:需要PostgreSQL,支持多实例(推荐生产)
  - DB-less模式:配置存YAML,适合简单场景

- [ ] **准备资源**
  ```bash
  # 最小配置(测试)
  Kong: 1 replica × 512MB = 512MB
  PostgreSQL: 1 replica × 512MB = 512MB
  总计: 1GB RAM, 1 vCPU
  
  # 生产配置
  Kong: 3 replica × 1GB = 3GB
  PostgreSQL: 2 replica × 2GB = 4GB
  总计: 7GB RAM, 4 vCPU
  ```

- [ ] **检查前置条件**
  ```bash
  kubectl version --short  # Kubernetes 1.20+
  helm version --short     # Helm 3.0+
  ```

##### **阶段2:部署PostgreSQL(1小时)**

```bash
# 使用Helm快速部署PostgreSQL
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

kubectl create namespace kong

helm install postgresql bitnami/postgresql \
  --namespace kong \
  --set auth.postgresPassword=strongpassword \
  --set auth.database=kong \
  --set primary.persistence.size=20Gi

# 等待PostgreSQL就绪
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=postgresql -n kong --timeout=180s
```

##### **阶段3:部署Kong(1小时)**

```bash
# 方案A:使用官方Helm Chart(推荐)
helm repo add kong https://charts.konghq.com
helm repo update

# 创建values.yaml
cat > kong-values.yaml <<EOF
env:
  database: postgres
  pg_host: postgresql.kong.svc
  pg_database: kong
  pg_user: postgres
  pg_password:
    valueFrom:
      secretKeyRef:
        name: postgresql
        key: postgres-password

proxy:
  enabled: true
  type: LoadBalancer
  http:
    enabled: true
    servicePort: 80
    containerPort: 8000
  tls:
    enabled: true
    servicePort: 443
    containerPort: 8443

admin:
  enabled: true
  type: ClusterIP
  http:
    enabled: true
    servicePort: 8001
    containerPort: 8001

ingressController:
  enabled: true  # 启用Kong Ingress Controller

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
EOF

# 安装Kong
helm install kong kong/kong -n kong -f kong-values.yaml

# 等待Kong就绪
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=kong -n kong --timeout=180s

# 获取Kong Proxy的外部IP
kubectl get svc kong-kong-proxy -n kong
```

**方案B:手动部署(完全控制)**

(参考Part 4的完整YAML配置)

##### **阶段4:安装deck CLI(15分钟)**

```bash
# macOS
brew install deck

# Linux
curl -sL https://github.com/kong/deck/releases/download/v1.28.0/deck_1.28.0_linux_amd64.tar.gz | tar xz
sudo mv deck /usr/local/bin/

# 验证
deck version
```

##### **阶段5:配置第一个API(1小时)**

```bash
# 1. Port-forward Kong Admin API(如果不在本地)
kubectl port-forward svc/kong-kong-admin -n kong 8001:8001 &

# 2. 创建kong.yaml配置文件
cat > kong.yaml <<EOF
_format_version: "3.0"

services:
  - name: httpbin-service
    url: http://httpbin.org
    routes:
      - name: httpbin-route
        paths:
          - /httpbin
        strip_path: true
    plugins:
      - name: rate-limiting
        config:
          minute: 10
EOF

# 3. 验证配置
deck validate --kong-addr http://localhost:8001 kong.yaml

# 4. 应用配置
deck sync --kong-addr http://localhost:8001 kong.yaml

# 5. 测试
KONG_PROXY_IP=$(kubectl get svc kong-kong-proxy -n kong -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$KONG_PROXY_IP/httpbin/get

# 应该返回httpbin的响应
```

---

#### **四、常用插件配置模板**

##### **1. JWT认证**

```yaml
plugins:
  - name: jwt
    config:
      claims_to_verify:
        - exp          # 验证过期时间
        - nbf          # 验证生效时间
      key_claim_name: iss
      secret_is_base64: false
      maximum_expiration: 31536000  # 最长1年
```

**生成JWT Token:**

```python
import jwt
import datetime

payload = {
    "iss": "your-consumer-key",
    "exp": datetime.datetime.utcnow() + datetime.timedelta(days=365),
    "sub": "user-id",
}
token = jwt.encode(payload, "your-secret", algorithm="HS256")
```

##### **2. 限流(多种策略)**

**策略A:全局限流**

```yaml
plugins:
  - name: rate-limiting
    config:
      second: 10
      minute: 100
      hour: 1000
      policy: local  # 单实例计数
```

**策略B:分布式限流(推荐生产)**

```yaml
plugins:
  - name: rate-limiting
    config:
      minute: 1000
      policy: redis  # 使用Redis,多实例共享
      redis_host: redis.default.svc
      redis_port: 6379
      redis_database: 0
      redis_timeout: 2000
      fault_tolerant: true  # Redis故障时不阻断
```

**策略C:按Consumer限流**

```yaml
consumers:
  - username: premium-user
    plugins:
      - name: rate-limiting
        config:
          minute: 10000  # Premium用户更高配额

  - username: free-user
    plugins:
      - name: rate-limiting
        config:
          minute: 100    # 免费用户较低配额
```

##### **3. CORS(跨域)**

```yaml
plugins:
  - name: cors
    config:
      origins:
        - "https://yourdomain.com"
        - "https://*.yourdomain.com"
      methods:
        - GET
        - POST
        - PUT
        - DELETE
        - OPTIONS
      headers:
        - Authorization
        - Content-Type
        - Accept
        - Origin
        - X-Requested-With
      exposed_headers:
        - X-Auth-Token
        - X-RateLimit-Remaining
      credentials: true
      max_age: 3600
      preflight_continue: false
```

##### **4. 请求/响应转换**

```yaml
plugins:
  # 添加Header
  - name: request-transformer
    config:
      add:
        headers:
          - "X-Service-ID:my-service"
          - "X-Request-Time:$(now())"
      remove:
        headers:
          - "X-Internal-Secret"  # 移除敏感Header
      
  # 重写路径
  - name: request-transformer-advanced
    config:
      replace:
        uri: "/v2$(uri)"  # /orders → /v2/orders
```

##### **5. IP白名单/黑名单**

```yaml
plugins:
  # 白名单(只允许特定IP)
  - name: ip-restriction
    config:
      allow:
        - 192.168.1.0/24
        - 10.0.0.1
  
  # 黑名单(阻止特定IP)
  - name: ip-restriction
    config:
      deny:
        - 123.45.67.89
```

---

#### **五、deck CLI常用命令**

```bash
# 1. 导出当前配置
deck dump --output-file kong-backup.yaml

# 2. 验证配置文件
deck validate kong.yaml

# 3. 预览变更(不实际应用)
deck diff kong.yaml

# 4. 应用配置
deck sync kong.yaml

# 5. 重置Kong(删除所有配置)
deck reset  # ⚠️ 危险操作!

# 6. 只同步特定workspace
deck sync --workspace my-workspace kong.yaml

# 7. 生成OpenAPI规范
deck file openapi2kong -s openapi.yaml -o kong.yaml
```

---

#### **六、监控和告警**

##### **1. 启用Prometheus插件**

```yaml
plugins:
  - name: prometheus
    config:
      per_consumer: true
      status_code_metrics: true
      latency_metrics: true
      bandwidth_metrics: true
```

##### **2. 关键指标告警**

```yaml
# Prometheus告警规则
groups:
- name: kong_alerts
  rules:
  # 高错误率
  - alert: KongHighErrorRate
    expr: |
      sum(rate(kong_http_requests_total{code=~"5.."}[5m])) 
      / 
      sum(rate(kong_http_requests_total[5m])) 
      > 0.05
    for: 5m
    annotations:
      summary: "Kong错误率过高(>5%)"
  
  # 高延迟
  - alert: KongHighLatency
    expr: |
      histogram_quantile(0.95, 
        sum(rate(kong_latency_bucket[5m])) by (le, service)
      ) > 5000
    for: 10m
    annotations:
      summary: "Kong P95延迟>5秒"
  
  # 限流拒绝激增
  - alert: KongRateLimitExceeded
    expr: |
      rate(kong_rate_limiting_exceeded_total[5m]) > 100
    for: 5m
    annotations:
      summary: "大量请求被限流拒绝"
```

---

#### **七、常见问题排查**

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| **502 Bad Gateway** | 上游服务不可达 | 检查Service的URL配置、上游服务健康状态 |
| **404 Not Found** | Route未匹配 | 检查paths、methods、hosts配置 |
| **401 Unauthorized** | 认证失败 | 检查JWT Token是否有效、iss字段是否匹配 |
| **429 Too Many Requests** | 达到限流阈值 | 检查rate-limiting配置、Consumer配额 |
| **Kong启动失败** | PostgreSQL连接失败 | 检查pg_host、pg_password配置 |
| **插件不生效** | 插件配置错误或未应用 | `deck sync`重新应用配置 |

**调试命令:**

```bash
# 检查Kong日志
kubectl logs -n kong kong-xxxx --tail=100

# 检查Admin API状态
curl http://localhost:8001/status

# 列出所有Service
curl http://localhost:8001/services

# 列出所有Route
curl http://localhost:8001/routes

# 检查特定Consumer
curl http://localhost:8001/consumers/your-consumer

# 测试Route匹配
curl -v http://$KONG_PROXY/your-path
```

---

#### **八、性能优化技巧**

##### **1. 使用缓存插件**

```yaml
plugins:
  - name: proxy-cache
    config:
      strategy: redis
      redis:
        host: redis.default.svc
        port: 6379
      content_type: ["application/json"]
      cache_ttl: 300
      cache_control: true
```

##### **2. 启用HTTP/2**

```yaml
# Kong配置
env:
  proxy_listen: 0.0.0.0:8000 http2, 0.0.0.0:8443 http2 ssl
```

##### **3. 调整worker数量**

```yaml
# Kong配置
env:
  nginx_worker_processes: auto  # 自动匹配CPU核心数
```

##### **4. 连接池优化**

```yaml
services:
  - name: my-service
    url: http://upstream:8080
    connect_timeout: 5000
    read_timeout: 60000
    write_timeout: 60000
    
    # 上游连接池
    client_certificate: null
    tls_verify: false
    retries: 3
```

---

#### **九、安全最佳实践**

- ✅ **使用强JWT密钥**:至少32字符随机字符串
- ✅ **启用HTTPS**:所有对外API必须使用TLS
- ✅ **定期轮换密钥**:每季度更换JWT secret
- ✅ **限制Admin API访问**:Admin API只能内网访问
- ✅ **启用IP白名单**:敏感API只允许特定IP访问
- ✅ **监控异常流量**:设置限流告警
- ✅ **最小权限原则**:每个Consumer只授予必要的Route权限

---

#### **十、成本估算**

**自托管Kong(中等规模,1000 RPS):**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
组件           规格              成本/月
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Kong          3 × 2GB RAM       $180
              1 vCPU            
PostgreSQL    2 × 2GB RAM       $120
              1 vCPU
负载均衡器     ALB/NLB           $30
Redis         1 × 1GB RAM       $40
              (限流/缓存)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
总计                            $370/月
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

vs. 托管API网关(阿里云 API Gateway):
  $3.50/百万请求 + $0.09/GB传输
  1000 RPS × 86400秒/天 × 30天 = 25.9亿请求/月
  成本: $9,065/月 (25倍贵!)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**结论:**
- ✅ 自托管Kong适合中大型项目(>100万请求/月)
- ✅ 托管服务适合小型项目(<100万请求/月)

</div>

---

## 第22章:关键要点

- **API网关不是奢侈品,是必需品**:当你有对外API时,网关提供的认证、限流、监控等功能不是"nice to have",而是"must have"。自己实现这些功能的成本远高于使用现成方案。

- **Nginx是反向代理,不是API管理平台**:Nginx擅长简单的HTTP路由和负载均衡,但缺乏精细化的API管理能力。不要试图用Lua脚本实现所有功能,这会让Nginx变得难以维护。

- **声明式配置优于命令式配置**:使用deck CLI的YAML配置,而不是直接调用Admin API。配置可以版本控制、Code Review、GitOps,极大提升团队协作效率。

- **Consumer是API管理的核心抽象**:为每个外部客户创建Consumer,可以实现per-consumer的限流、配额、认证。这比IP级别的限流灵活得多。

- **插件生态是Kong的最大优势**:1800+插件覆盖几乎所有需求,从简单的限流到复杂的协议转换。优先使用现有插件,只在真正需要时才开发自定义插件。

- **监控Kong自身同样重要**:Kong是所有API请求的单点入口,它的健康状态直接影响业务。使用Prometheus监控Kong的请求率、错误率、延迟,设置告警。

- **API可以成为新的收入来源**:不要把API只看作技术工具。通过Kong的统计数据,你可以实现基于使用量的计费,将API变成一条新的商业化路径。

---

