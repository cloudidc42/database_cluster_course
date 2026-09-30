# Part 55: Service Mesh (Istio/Linkerd) สำหรับ Database Cluster

## บทนำ

Service Mesh เป็น infrastructure layer ที่ทำหน้าที่จัดการ network traffic ระหว่าง microservices อย่างอัตโนมัติ สำหรับ database cluster การใช้ Service Mesh ช่วยในเรื่องของ mTLS encryption, traffic management, observability และ circuit breaking โดยไม่ต้องเขียน code เพิ่มใน application บทนี้จะครอบคลุม Istio และ Linkerd สำหรับ database cluster โดยเฉพาะ

---

## 1. Service Mesh: คืออะไร ทำไมต้องใช้

### 1.1 ปัญหาที่ Service Mesh แก้ไข

```
ก่อน Service Mesh (แบบเก่า):
┌──────────────┐    TCP    ┌──────────────┐
│  Application │ ────────▶ │   Database   │
│  (ต้องเขียน  │           │              │
│  retry/tls/  │           │              │
│  circuit     │           │              │
│  breaker เอง)│           │              │
└──────────────┘           └──────────────┘

หลัง Service Mesh:
┌──────────────┐     ┌─────────┐    mTLS   ┌─────────┐    ┌──────────────┐
│  Application │────▶│  Sidecar│ ─────────▶│  Sidecar│───▶│   Database   │
│  (เขียนแค่   │     │ (Envoy) │           │ (Envoy) │    │              │
│  business    │     └─────────┘           └─────────┘    └──────────────┘
│  logic)      │          ▲                     ▲
└──────────────┘          │                     │
                    ┌─────┴─────────────────────┴─────┐
                    │         Control Plane            │
                    │          (istiod)                │
                    └──────────────────────────────────┘
```

### 1.2 Service Mesh Capabilities

```
Traffic Management:
  - Load balancing (round-robin, least-connection, random)
  - Traffic splitting (canary, A/B testing)
  - Retries, timeouts, circuit breaking
  - Request routing based on headers/paths
  - Traffic mirroring

Security:
  - Mutual TLS (mTLS) ระหว่างทุก services
  - Certificate management อัตโนมัติ
  - Authorization policies
  - Service-to-service authentication

Observability:
  - Distributed tracing (Jaeger, Zipkin)
  - Metrics (request rate, error rate, latency)
  - Service topology visualization (Kiali)
  - Access logs

Resilience:
  - Circuit breaker
  - Retry policies
  - Timeout policies
  - Rate limiting
```

---

## 2. Istio: Installation และ Configuration

### 2.1 Istio Architecture

```
Control Plane (istiod):
  - Pilot:    Traffic management (distributes Envoy configs)
  - Citadel:  Certificate management (mTLS)
  - Galley:   Configuration validation

Data Plane (Envoy Sidecars):
  - Interceptor inbound/outbound traffic
  - ทุก Pod ได้ sidecar โดยอัตโนมัติ (ถ้า namespace มี label)

External Components:
  - Kiali:   Visualization dashboard
  - Jaeger:  Distributed tracing
  - Prometheus: Metrics collection
  - Grafana: Metrics visualization
```

### 2.2 ติดตั้ง Istio

```bash
# Download istioctl
curl -L https://istio.io/downloadIstio | sh -

# เพิ่ม istioctl ใน PATH
export PATH=$PWD/istio-1.20.0/bin:$PATH

# ตรวจสอบ
istioctl version

# ตรวจสอบ prerequisites
istioctl x precheck

# Install Istio (default profile)
istioctl install --set profile=default -y

# Install Istio (production profile - HA control plane)
istioctl install --set profile=default \
  --set components.pilot.k8s.replicaCount=2 \
  --set components.ingressGateway.k8s.replicaCount=2 \
  -y

# ตรวจสอบ installation
kubectl get pods -n istio-system
kubectl get svc -n istio-system

# Enable sidecar injection สำหรับ namespace
kubectl label namespace databases istio-injection=enabled
kubectl label namespace applications istio-injection=enabled
```

### 2.3 IstioOperator Configuration

```yaml
# istio-operator.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-control-plane
spec:
  profile: default
  
  components:
    # Control Plane HA
    pilot:
      k8s:
        replicaCount: 2
        resources:
          requests:
            cpu: 500m
            memory: 2Gi
          limits:
            cpu: 1000m
            memory: 4Gi
        hpaSpec:
          minReplicas: 2
          maxReplicas: 5
          
    # Ingress Gateway
    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      k8s:
        replicaCount: 2
        service:
          type: LoadBalancer
          ports:
          - port: 80
            targetPort: 8080
            name: http2
          - port: 443
            targetPort: 8443
            name: https
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 2000m
            memory: 1Gi
            
  meshConfig:
    # เปิด access logging
    accessLogFile: /dev/stdout
    accessLogEncoding: JSON
    
    # Default retry policy
    defaultConfig:
      tracing:
        sampling: 1.0  # 1% sampling ใน production
      
    # Enable distributed tracing
    enableTracing: true
    
    # mTLS settings
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY  # ไม่อนุญาต traffic ไป unknown services

  values:
    global:
      # Pilot settings
      pilotCertProvider: istiod
      
      # Proxy settings
      proxy:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 2000m
            memory: 1Gi
        logLevel: warning
```

```bash
# Apply IstioOperator
istioctl install -f istio-operator.yaml -y

# Verify
istioctl verify-install
```

### 2.4 VirtualService

VirtualService กำหนด routing rules สำหรับ traffic:

```yaml
# virtualservice-app.yaml
---
# VirtualService สำหรับ Application
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: nodejs-app
  namespace: applications
spec:
  hosts:
  - nodejs-app
  - api.example.com
  
  # กำหนด gateways ที่ใช้
  gateways:
  - istio-system/ingressgateway
  - mesh  # internal traffic
  
  http:
  # ─── Version-based routing ───────────────────────────
  # Traffic 90% ไป v1, 10% ไป v2 (canary)
  - name: canary-routing
    match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: nodejs-app
        subset: v2
      weight: 100
      
  # ─── Default routing ─────────────────────────────────
  - name: default-routing
    route:
    - destination:
        host: nodejs-app
        subset: v1
      weight: 90
    - destination:
        host: nodejs-app
        subset: v2
      weight: 10
      
    # Retry policy
    retries:
      attempts: 3
      perTryTimeout: 5s
      retryOn: 5xx,retriable-4xx
      
    # Timeout
    timeout: 30s
    
    # Fault injection (สำหรับ testing)
    # fault:
    #   delay:
    #     percentage:
    #       value: 5.0
    #     fixedDelay: 5s

---
# VirtualService สำหรับ PostgreSQL (Read/Write Splitting)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: postgresql-routing
  namespace: databases
spec:
  hosts:
  - postgresql
  
  tcp:
  - match:
    - port: 5432
    route:
    - destination:
        host: postgresql-primary
        port:
          number: 5432
```

### 2.5 DestinationRule

DestinationRule กำหนด load balancing, circuit breaking และ connection pool:

```yaml
# destinationrule-app.yaml
---
# DestinationRule สำหรับ Application
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: nodejs-app
  namespace: applications
spec:
  host: nodejs-app
  
  trafficPolicy:
    # Load balancing
    loadBalancer:
      simple: LEAST_CONN  # ROUND_ROBIN, LEAST_CONN, RANDOM, PASSTHROUGH
      
    # Connection pool
    connectionPool:
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
        maxRetries: 3
        idleTimeout: 90s
        h2UpgradePolicy: UPGRADE
      tcp:
        maxConnections: 100
        connectTimeout: 30ms
        tcpKeepalive:
          time: 7200s
          interval: 75s
          
    # Circuit Breaker (Outlier Detection)
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 30
      
    # mTLS
    tls:
      mode: ISTIO_MUTUAL
      
  # Subset definitions (สำหรับ canary)
  subsets:
  - name: v1
    labels:
      version: "v1"
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
        
  - name: v2
    labels:
      version: "v2"
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN

---
# DestinationRule สำหรับ PostgreSQL
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: postgresql
  namespace: databases
spec:
  host: "*.databases.svc.cluster.local"
  
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 5s
        tcpKeepalive:
          time: 7200s
          interval: 75s
          
    # Circuit Breaker สำหรับ Database
    outlierDetection:
      consecutiveLocalOriginFailures: 3
      interval: 30s
      baseEjectionTime: 60s
      maxEjectionPercent: 100
      
    # mTLS
    tls:
      mode: ISTIO_MUTUAL

---
# DestinationRule สำหรับ Redis
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: redis
  namespace: databases
spec:
  host: redis-master.databases.svc.cluster.local
  
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 200
        connectTimeout: 3s
    outlierDetection:
      consecutiveLocalOriginFailures: 5
      interval: 10s
      baseEjectionTime: 30s
    tls:
      mode: ISTIO_MUTUAL
```

### 2.6 Gateway

```yaml
# gateway.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: main-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  # HTTPS
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: api-tls-credential  # จาก K8s Secret
    hosts:
    - api.example.com
    
  # HTTP (redirect to HTTPS)
  - port:
      number: 80
      name: http
      protocol: HTTP
    tls:
      httpsRedirect: true
    hosts:
    - api.example.com
```

### 2.7 PeerAuthentication (mTLS)

```yaml
# peer-authentication.yaml
---
# Enable STRICT mTLS สำหรับทุก services ใน databases namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default-mtls
  namespace: databases
spec:
  mtls:
    mode: STRICT  # ต้องใช้ mTLS เสมอ

---
# PERMISSIVE mode สำหรับ applications (allow non-mTLS ด้วย)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: app-mtls
  namespace: applications
spec:
  mtls:
    mode: PERMISSIVE  # รับทั้ง mTLS และ plain text

---
# Port-level mTLS policy
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: postgresql-mtls
  namespace: databases
spec:
  selector:
    matchLabels:
      app: postgresql
  mtls:
    mode: STRICT
  portLevelMtls:
    5432:
      mode: STRICT
    9187:  # metrics port
      mode: PERMISSIVE
```

### 2.8 AuthorizationPolicy

```yaml
# authorization-policies.yaml
---
# อนุญาตเฉพาะ application namespace เข้าถึง PostgreSQL
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-app-to-postgresql
  namespace: databases
spec:
  selector:
    matchLabels:
      app: postgresql
  action: ALLOW
  rules:
  - from:
    - source:
        namespaces: ["applications"]
        principals: 
        - "cluster.local/ns/applications/sa/app-service-account"
    to:
    - operation:
        ports: ["5432"]

---
# Deny all ถ้าไม่มี rule match (ใน databases namespace)
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: databases
spec:
  {}  # Empty spec = deny all

---
# อนุญาต monitoring namespace เข้าถึง metrics
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: databases
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        namespaces: ["monitoring"]
    to:
    - operation:
        ports: ["9187", "9121"]  # PostgreSQL and Redis exporters

---
# อนุญาต internal namespace traffic
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-internal
  namespace: applications
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        namespaces: ["applications", "ingress-system"]
```

---

## 3. Linkerd: Lighter Weight Alternative

### 3.1 Linkerd vs Istio

| Feature | Linkerd | Istio |
|---------|---------|-------|
| **Complexity** | ต่ำ | สูง |
| **Resource usage** | ต่ำมาก | สูง |
| **mTLS** | อัตโนมัติ | ต้อง configure |
| **Proxy** | Linkerd2-proxy (Rust) | Envoy (C++) |
| **Learning curve** | ง่าย | ยาก |
| **Features** | พื้นฐาน | ครบครัน |
| **Protocol support** | HTTP/gRPC/TCP | ทุกอย่าง |

### 3.2 ติดตั้ง Linkerd

```bash
# ติดตั้ง CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
export PATH=$HOME/.linkerd2/bin:$PATH

# ตรวจสอบ
linkerd version
linkerd check --pre  # pre-installation check

# Install Linkerd control plane
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -

# ตรวจสอบ
linkerd check

# Install viz (observability)
linkerd viz install | kubectl apply -f -
linkerd viz check

# Open dashboard
linkerd viz dashboard
```

### 3.3 Inject Linkerd Proxy

```bash
# Inject ใน namespace (automatic)
kubectl annotate namespace applications linkerd.io/inject=enabled
kubectl annotate namespace databases linkerd.io/inject=enabled

# Inject ใน deployment เฉพาะ
kubectl get deploy nodejs-app -n applications -o yaml | \
  linkerd inject - | \
  kubectl apply -f -

# ตรวจสอบ injection
linkerd check --proxy -n applications

# ดู proxy status
linkerd viz stat deployments -n applications
linkerd viz top deploy/nodejs-app -n applications
```

### 3.4 Linkerd ServiceProfile

```yaml
# serviceprofile-postgresql.yaml
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: postgresql.databases.svc.cluster.local
  namespace: databases
spec:
  routes:
  # กำหนด retry policy สำหรับ specific operations
  - name: "SELECT queries"
    condition:
      method: POST
      pathRegex: "/query"
    isRetryable: true
    timeout: 30s
    
  # Retry policy
  retryBudget:
    retryRatio: 0.2      # retry ได้ไม่เกิน 20% ของ requests
    minRetriesPerSecond: 10
    ttl: 10s
```

### 3.5 Linkerd Traffic Split (Canary)

```yaml
# traffic-split-canary.yaml
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: nodejs-app-split
  namespace: applications
spec:
  service: nodejs-app
  backends:
  - service: nodejs-app-stable
    weight: "900m"   # 90%
  - service: nodejs-app-canary
    weight: "100m"   # 10%
```

---

## 4. Traffic Mirroring (Shadow Traffic)

```yaml
# traffic-mirror.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: postgresql-mirror
  namespace: databases
spec:
  hosts:
  - postgresql-primary
  
  tcp:
  - match:
    - port: 5432
    route:
    - destination:
        host: postgresql-primary
        port:
          number: 5432
      weight: 100
    # Mirror traffic ไปยัง staging cluster
    mirror:
      host: postgresql-staging
      port:
        number: 5432
    mirrorPercentage:
      value: 10.0  # Mirror 10% ของ traffic
```

---

## 5. Canary Deployments

### 5.1 Gradual Traffic Shifting

```yaml
# Step 1: เริ่มด้วย 5% traffic ไป v2
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: nodejs-app
  namespace: applications
spec:
  hosts:
  - nodejs-app
  http:
  - route:
    - destination:
        host: nodejs-app
        subset: v1
      weight: 95
    - destination:
        host: nodejs-app
        subset: v2
      weight: 5
```

```bash
# Monitor error rates ระหว่าง canary
kubectl -n monitoring exec -it prometheus-0 -- \
  promtool query instant 'rate(istio_requests_total{destination_service="nodejs-app",response_code=~"5.*"}[5m])'

# ถ้า OK เพิ่มเป็น 20%
kubectl patch virtualservice nodejs-app -n applications --type=merge \
  -p '{"spec":{"http":[{"route":[{"destination":{"host":"nodejs-app","subset":"v1"},"weight":80},{"destination":{"host":"nodejs-app","subset":"v2"},"weight":20}]}]}}'

# ถ้า OK เพิ่มเป็น 50%
# ถ้า OK เพิ่มเป็น 100% (full rollout)
```

---

## 6. Observability

### 6.1 Kiali Dashboard

```bash
# ติดตั้ง Kiali
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/kiali.yaml

# เปิด dashboard
istioctl dashboard kiali
# หรือ
kubectl port-forward svc/kiali 20001:20001 -n istio-system
```

Kiali แสดง:
- Service graph (topology)
- Traffic flow
- Error rates
- Response times
- mTLS status

### 6.2 Jaeger Distributed Tracing

```bash
# ติดตั้ง Jaeger
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/jaeger.yaml

# เปิด dashboard
istioctl dashboard jaeger
```

### 6.3 Prometheus + Grafana

```bash
# ติดตั้ง Prometheus
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/prometheus.yaml

# ติดตั้ง Grafana
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/grafana.yaml

# เปิด Grafana
istioctl dashboard grafana
```

### 6.4 Metrics ที่สำคัญ

```promql
# ─── Request Rate ───────────────────────────────────────
# Requests per second
rate(istio_requests_total{destination_service_name="nodejs-app"}[5m])

# ─── Error Rate ─────────────────────────────────────────
# Error rate (5xx)
sum(rate(istio_requests_total{
  destination_service_name="nodejs-app",
  response_code=~"5.*"
}[5m])) / 
sum(rate(istio_requests_total{
  destination_service_name="nodejs-app"
}[5m]))

# ─── Latency ────────────────────────────────────────────
# P50 latency
histogram_quantile(0.50, 
  sum(rate(istio_request_duration_milliseconds_bucket{
    destination_service_name="nodejs-app"
  }[5m])) by (le)
)

# P99 latency
histogram_quantile(0.99,
  sum(rate(istio_request_duration_milliseconds_bucket{
    destination_service_name="nodejs-app"
  }[5m])) by (le)
)

# ─── Database Metrics ────────────────────────────────────
# PostgreSQL connection rate through service mesh
rate(istio_requests_total{
  destination_service_name="postgresql-primary"
}[5m])

# TCP bytes transferred to PostgreSQL
rate(istio_tcp_sent_bytes_total{
  destination_service_name="postgresql-primary"
}[5m])
```

---

## 7. Full Istio Configuration สำหรับ Database Cluster

### 7.1 Complete Setup

```bash
#!/bin/bash
# setup-istio.sh

set -e

echo "=== Setting up Istio for Database Cluster ==="

# ─── Install Istio ──────────────────────────────────────
echo "1. Installing Istio..."
istioctl install --set profile=default \
  --set components.pilot.k8s.replicaCount=2 \
  -y

echo "2. Installing addons..."
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/grafana.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/kiali.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/jaeger.yaml

# ─── Enable namespaces ──────────────────────────────────
echo "3. Enabling sidecar injection..."
kubectl label namespace databases istio-injection=enabled --overwrite
kubectl label namespace applications istio-injection=enabled --overwrite

# ─── Apply configurations ────────────────────────────────
echo "4. Applying security policies..."
kubectl apply -f peer-authentication.yaml
kubectl apply -f authorization-policies.yaml

echo "5. Applying traffic management..."
kubectl apply -f destinationrule-app.yaml
kubectl apply -f virtualservice-app.yaml
kubectl apply -f gateway.yaml

# ─── Restart pods to inject sidecars ────────────────────
echo "6. Restarting pods for sidecar injection..."
kubectl rollout restart deployment -n applications
kubectl rollout restart statefulset -n databases

# ─── Verify ─────────────────────────────────────────────
echo "7. Verifying installation..."
sleep 30
istioctl verify-install
linkerd check 2>/dev/null || true

echo "=== Setup Complete ==="
echo "Access dashboards:"
echo "  Kiali:    istioctl dashboard kiali"
echo "  Grafana:  istioctl dashboard grafana"
echo "  Jaeger:   istioctl dashboard jaeger"
```

### 7.2 Complete YAML Manifest

```yaml
# istio-database-cluster.yaml

---
# ─── Global mTLS ────────────────────────────────────────────
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT

---
# ─── Application Destination Rule ───────────────────────────
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: nodejs-app
  namespace: applications
spec:
  host: nodejs-app
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
    connectionPool:
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
      tcp:
        maxConnections: 100
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2

---
# ─── PostgreSQL DestinationRule ─────────────────────────────
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: postgresql
  namespace: databases
spec:
  host: postgresql-primary.databases.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 5s
    outlierDetection:
      consecutiveLocalOriginFailures: 3
      interval: 30s
      baseEjectionTime: 60s
    tls:
      mode: ISTIO_MUTUAL

---
# ─── Application VirtualService ─────────────────────────────
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: nodejs-app
  namespace: applications
spec:
  hosts:
  - nodejs-app
  - api.example.com
  gateways:
  - istio-system/main-gateway
  - mesh
  http:
  - match:
    - headers:
        x-version:
          exact: v2
    route:
    - destination:
        host: nodejs-app
        subset: v2
  - route:
    - destination:
        host: nodejs-app
        subset: v1
      weight: 100
    retries:
      attempts: 3
      perTryTimeout: 5s
      retryOn: 5xx,retriable-4xx
    timeout: 30s

---
# ─── Authorization: Allow app → postgresql ──────────────────
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-app-to-postgresql
  namespace: databases
spec:
  selector:
    matchLabels:
      app: postgresql
  action: ALLOW
  rules:
  - from:
    - source:
        namespaces: ["applications"]
  to:
  - operation:
      ports: ["5432"]

---
# ─── Authorization: Allow app → redis ───────────────────────
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-app-to-redis
  namespace: databases
spec:
  selector:
    matchLabels:
      app: redis
  action: ALLOW
  rules:
  - from:
    - source:
        namespaces: ["applications"]
  to:
  - operation:
      ports: ["6379"]

---
# ─── Authorization: Allow monitoring ────────────────────────
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-monitoring
  namespace: databases
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        namespaces: ["monitoring", "istio-system"]
  to:
  - operation:
      ports: ["9187", "9121", "9090"]

---
# ─── Gateway ────────────────────────────────────────────────
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: main-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: api-tls
    hosts:
    - api.example.com
  - port:
      number: 80
      name: http
      protocol: HTTP
    tls:
      httpsRedirect: true
    hosts:
    - api.example.com
```

---

## 8. Troubleshooting

### 8.1 Debug Commands

```bash
# ตรวจสอบ sidecar injection
kubectl get pod nodejs-app-xxxxx -n applications -o jsonpath='{.spec.containers[*].name}'
# ต้องเห็น: nodejs-app istio-proxy

# ดู proxy logs
kubectl logs nodejs-app-xxxxx -c istio-proxy -n applications

# ตรวจสอบ mTLS
istioctl authn tls-check nodejs-app.applications.svc.cluster.local

# ดู Envoy configuration
istioctl proxy-config cluster nodejs-app-xxxxx -n applications
istioctl proxy-config listener nodejs-app-xxxxx -n applications
istioctl proxy-config route nodejs-app-xxxxx -n applications

# ตรวจสอบ authorization policies
istioctl experimental authz check nodejs-app-xxxxx -n applications

# Debug traffic
istioctl dashboard envoy nodejs-app-xxxxx -n applications

# ตรวจสอบ certificates
istioctl proxy-config secret nodejs-app-xxxxx -n applications
```

### 8.2 Common Issues

```bash
# Issue: Pods ไม่ได้ sidecar injection
# Solution: ตรวจสอบ namespace label
kubectl get namespace applications --show-labels | grep istio-injection

# Issue: mTLS failing
# Solution: ตรวจสอบ PeerAuthentication
kubectl get peerauthentication -A
istioctl authn tls-check --all-namespaces

# Issue: Connection refused ไปยัง database
# Solution: ตรวจสอบ AuthorizationPolicy
kubectl get authorizationpolicy -n databases
istioctl analyze -n databases

# Issue: Circuit breaker กำลัง open
# Solution: ดู Envoy stats
kubectl exec nodejs-app-xxxxx -n applications -c istio-proxy -- \
  pilot-agent request GET stats | grep outlier

# Issue: Slow performance หลัง mesh
# Solution: ลด sampling rate และตรวจสอบ resource usage
kubectl top pods -n istio-system
```

---

## สรุป

Service Mesh ให้ความสามารถที่ทรงพลังสำหรับ database cluster:

1. **mTLS** ป้องกัน eavesdropping ระหว่าง app และ database
2. **AuthorizationPolicy** กำหนดว่า service ใดเข้าถึง database ได้
3. **Circuit Breaking** ป้องกัน cascade failures
4. **Traffic Management** รองรับ canary deployments
5. **Observability** เห็น request flow ทั้งหมดใน mesh

### คำแนะนำ

| Scenario | Solution |
|----------|----------|
| Simple HA + mTLS | Linkerd |
| Full traffic management | Istio |
| Database protection | Istio + AuthorizationPolicy |
| Canary deployments | Istio VirtualService |
| Observability | Kiali + Jaeger + Prometheus |

```bash
# Quick start
istioctl install --set profile=default -y
kubectl label namespace databases istio-injection=enabled
kubectl apply -f istio-database-cluster.yaml
istioctl dashboard kiali
```
