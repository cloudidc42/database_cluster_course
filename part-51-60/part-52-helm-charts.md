# Part 52: Helm Charts สำหรับ Database Cluster

## บทนำ

Helm เป็น package manager สำหรับ Kubernetes ที่ช่วยให้การ deploy applications บน Kubernetes ง่ายขึ้นอย่างมาก แทนที่จะต้องจัดการ YAML files หลายสิบไฟล์ด้วยตัวเอง Helm ช่วยให้เราสามารถ package, configure และ deploy applications ได้อย่างมีระบบ บทนี้จะครอบคลุมการใช้ Helm สำหรับ database cluster อย่างครบถ้วน

---

## 1. Helm Concepts

### 1.1 Helm Architecture

```
┌─────────────────────────────────────────┐
│              Helm Client                │
│         (helm CLI tool)                 │
└────────────────┬────────────────────────┘
                 │ kubectl API
                 ▼
┌─────────────────────────────────────────┐
│           Kubernetes API                │
│                                         │
│  ┌──────────┐  ┌──────────┐  ┌───────┐ │
│  │ Release  │  │ Manifest │  │Config │ │
│  │ History  │  │  Store   │  │  Map  │ │
│  └──────────┘  └──────────┘  └───────┘ │
└─────────────────────────────────────────┘
```

### 1.2 Helm Terminology

| คำศัพท์ | ความหมาย |
|---------|---------|
| **Chart** | Package ที่ประกอบด้วย Kubernetes manifests templates |
| **Release** | Instance ของ chart ที่ deploy แล้วบน K8s |
| **Repository** | ที่เก็บ charts (เหมือน npm registry) |
| **Values** | Configuration ที่ใส่ลงใน chart templates |
| **Revision** | Version history ของ release |

### 1.3 Chart Structure

```
mychart/
├── Chart.yaml          # ข้อมูล metadata ของ chart
├── values.yaml         # Default configuration values
├── values.schema.json  # JSON Schema สำหรับ validate values
├── charts/             # Sub-charts (dependencies)
├── crds/               # Custom Resource Definitions
├── templates/          # Kubernetes manifests templates
│   ├── NOTES.txt       # ข้อความแสดงหลัง install
│   ├── _helpers.tpl    # Helper functions
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   └── tests/
│       └── test-connection.yaml
└── .helmignore         # ไฟล์ที่ไม่ต้องการ include
```

---

## 2. Installation และ Setup

### 2.1 ติดตั้ง Helm

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Windows (Chocolatey)
choco install kubernetes-helm

# ตรวจสอบ version
helm version
# Output: version.BuildInfo{Version:"v3.14.0", ...}
```

### 2.2 Helm Repository Management

```bash
# เพิ่ม repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable https://charts.helm.sh/stable
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add cert-manager https://charts.jetstack.io
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add minio https://charts.min.io/

# อัปเดต repositories
helm repo update

# ค้นหา charts
helm search repo postgresql
helm search repo redis
helm search repo minio

# ดู chart versions ทั้งหมด
helm search repo bitnami/postgresql --versions

# ดู chart information
helm show chart bitnami/postgresql
helm show values bitnami/postgresql
helm show readme bitnami/postgresql
```

---

## 3. Installing Database Charts

### 3.1 PostgreSQL ด้วย Bitnami

```bash
# ดู values ทั้งหมด
helm show values bitnami/postgresql > postgresql-default-values.yaml
```

สร้างไฟล์ values:
```yaml
# postgresql-values.yaml
global:
  postgresql:
    auth:
      postgresPassword: "P@ssw0rd#2024"
      username: "appuser"
      password: "App#P@ss2024"
      database: "appdb"
    service:
      ports:
        postgresql: 5432

architecture: replication

primary:
  name: primary
  
  persistence:
    enabled: true
    storageClass: "fast-ssd"
    size: 50Gi
    
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "4Gi"
      cpu: "2000m"
      
  configuration: |
    max_connections = 300
    shared_buffers = 512MB
    effective_cache_size = 2GB
    work_mem = 8MB
    maintenance_work_mem = 128MB
    wal_level = replica
    max_wal_senders = 10
    wal_keep_size = 2GB
    
  extendedConfiguration: |
    log_min_duration_statement = 1000
    log_checkpoints = on
    track_io_timing = on
    
  livenessProbe:
    enabled: true
    initialDelaySeconds: 60
    periodSeconds: 10
    timeoutSeconds: 5
    successThreshold: 1
    failureThreshold: 3
    
  readinessProbe:
    enabled: true
    initialDelaySeconds: 30
    periodSeconds: 5
    
  podAntiAffinityPreset: hard

readReplicas:
  name: read
  replicaCount: 2
  
  persistence:
    enabled: true
    storageClass: "fast-ssd"
    size: 50Gi
    
  resources:
    requests:
      memory: "512Mi"
      cpu: "250m"
    limits:
      memory: "2Gi"
      cpu: "1000m"
      
  podAntiAffinityPreset: soft

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: monitoring
    
service:
  primary:
    type: ClusterIP
    ports:
      postgresql: 5432
  readReplica:
    type: ClusterIP
    ports:
      postgresql: 5432
```

```bash
# Install PostgreSQL
helm install postgresql bitnami/postgresql \
  --namespace databases \
  --create-namespace \
  -f postgresql-values.yaml \
  --version 13.4.0

# ดู status
helm status postgresql -n databases

# ดู release notes
helm get notes postgresql -n databases
```

### 3.2 Redis Cluster ด้วย Bitnami

```yaml
# redis-values.yaml
architecture: replication

auth:
  enabled: true
  password: "Red!s#P@ss2024"
  
sentinel:
  enabled: true
  masterSet: "mymaster"
  quorum: 2
  
master:
  count: 1
  
  persistence:
    enabled: true
    storageClass: "fast-ssd"
    size: 10Gi
    
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "4Gi"
      cpu: "1000m"
      
  configuration: |
    maxmemory 2gb
    maxmemory-policy allkeys-lru
    appendonly yes
    appendfsync everysec
    
replica:
  replicaCount: 2
  
  persistence:
    enabled: true
    storageClass: "fast-ssd"
    size: 10Gi
    
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "4Gi"
      cpu: "1000m"

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: monitoring
```

```bash
# Install Redis
helm install redis bitnami/redis \
  --namespace databases \
  -f redis-values.yaml

# ดู pods
kubectl get pods -n databases -l app.kubernetes.io/name=redis
```

### 3.3 MinIO

```yaml
# minio-values.yaml
mode: distributed
replicas: 4

auth:
  rootUser: "minioadmin"
  rootPassword: "Min!o#P@ss2024"
  
persistence:
  enabled: true
  storageClass: "fast-ssd"
  size: 100Gi
  
resources:
  requests:
    memory: "1Gi"
    cpu: "500m"
  limits:
    memory: "8Gi"
    cpu: "4000m"

ingress:
  enabled: true
  hostname: minio-api.example.com
  
consoleIngress:
  enabled: true
  hostname: minio-console.example.com

metrics:
  serviceMonitor:
    enabled: true
    namespace: monitoring
```

```bash
helm install minio minio/minio \
  --namespace databases \
  -f minio-values.yaml
```

---

## 4. Helm Commands

### 4.1 Lifecycle Commands

```bash
# Install
helm install RELEASE_NAME CHART [flags]
helm install postgresql bitnami/postgresql -n databases -f values.yaml

# Upgrade
helm upgrade RELEASE_NAME CHART [flags]
helm upgrade postgresql bitnami/postgresql -n databases -f values.yaml

# Install or Upgrade (atomic)
helm upgrade --install postgresql bitnami/postgresql \
  -n databases \
  --create-namespace \
  -f values.yaml \
  --wait \
  --timeout 10m

# Rollback
helm rollback RELEASE_NAME [REVISION] [flags]
helm rollback postgresql -n databases
helm rollback postgresql 2 -n databases  # rollback ไปยัง revision 2

# Uninstall
helm uninstall postgresql -n databases
helm uninstall postgresql -n databases --keep-history  # เก็บ history

# List releases
helm list -n databases
helm list -A  # ทุก namespaces
helm list -n databases --all  # รวม failed/pending

# Get release info
helm get all postgresql -n databases
helm get values postgresql -n databases
helm get manifest postgresql -n databases
helm get hooks postgresql -n databases
helm get notes postgresql -n databases

# History
helm history postgresql -n databases
```

### 4.2 Template Commands

```bash
# Render templates (ไม่ deploy)
helm template postgresql bitnami/postgresql -f values.yaml

# Render template เฉพาะบางไฟล์
helm template postgresql bitnami/postgresql \
  -f values.yaml \
  --show-only templates/statefulset.yaml

# Lint chart
helm lint ./mychart
helm lint ./mychart -f values-prod.yaml

# Package chart
helm package ./mychart
helm package ./mychart --destination ./packages

# Verify chart
helm verify ./mychart-1.0.0.tgz
```

---

## 5. สร้าง Custom Helm Chart

### 5.1 สร้าง Chart ใหม่

```bash
# สร้าง chart skeleton
helm create database-app

# ดู structure
tree database-app/
```

### 5.2 Chart.yaml

```yaml
# database-app/Chart.yaml
apiVersion: v2
name: database-app
description: "Node.js Application พร้อม PostgreSQL, Redis และ MinIO"
type: application
version: 1.0.0
appVersion: "1.0.0"
keywords:
  - nodejs
  - postgresql
  - redis
  - minio
home: https://github.com/mycompany/database-app
sources:
  - https://github.com/mycompany/database-app
maintainers:
  - name: DevOps Team
    email: devops@mycompany.com
    url: https://mycompany.com
icon: https://example.com/icon.png
annotations:
  category: Application

# Dependencies
dependencies:
  - name: postgresql
    version: "13.4.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
    tags:
      - database
      
  - name: redis
    version: "18.6.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
    tags:
      - cache
```

### 5.3 values.yaml

```yaml
# database-app/values.yaml

# Global settings
global:
  imageRegistry: ""
  imagePullSecrets: []
  storageClass: ""

# Application settings
app:
  image:
    repository: myregistry.io/nodejs-app
    tag: "1.0.0"
    pullPolicy: IfNotPresent
    
  replicaCount: 3
  
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "1Gi"
      cpu: "500m"
      
  service:
    type: ClusterIP
    port: 3000
    
  ingress:
    enabled: true
    className: nginx
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
    hosts:
    - host: api.example.com
      paths:
      - path: /
        pathType: Prefix
    tls:
    - secretName: api-tls
      hosts:
      - api.example.com
      
  hpa:
    enabled: true
    minReplicas: 3
    maxReplicas: 20
    targetCPUUtilizationPercentage: 70
    targetMemoryUtilizationPercentage: 80
    
  pdb:
    enabled: true
    minAvailable: 2
    
  env:
    NODE_ENV: production
    LOG_LEVEL: info
    DB_POOL_MIN: "5"
    DB_POOL_MAX: "20"
    DB_POOL_IDLE: "10000"
    
  envFromSecrets: []

# Database settings (uses bitnami/postgresql subchart)
postgresql:
  enabled: true
  auth:
    postgresPassword: ""  # จะถูก generate ถ้าว่าง
    username: appuser
    password: ""
    database: appdb
  primary:
    persistence:
      enabled: true
      size: 50Gi
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
      limits:
        memory: "4Gi"
        cpu: "2000m"
  readReplicas:
    replicaCount: 2
    persistence:
      enabled: true
      size: 50Gi
    resources:
      requests:
        memory: "512Mi"
        cpu: "250m"
      limits:
        memory: "2Gi"
        cpu: "1000m"
  metrics:
    enabled: true

# Redis settings (uses bitnami/redis subchart)
redis:
  enabled: true
  architecture: replication
  auth:
    enabled: true
    password: ""
  master:
    persistence:
      enabled: true
      size: 10Gi
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "4Gi"
        cpu: "1000m"
  replica:
    replicaCount: 2
    persistence:
      enabled: true
      size: 10Gi
  metrics:
    enabled: true

# MinIO settings
minio:
  enabled: true
  auth:
    rootUser: minioadmin
    rootPassword: ""
  persistence:
    enabled: true
    size: 100Gi
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "8Gi"
      cpu: "4000m"

# Monitoring
monitoring:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: monitoring

# Secrets (สร้างใน install time)
secrets:
  create: true
  
# Namespace
namespaceOverride: ""
```

### 5.4 _helpers.tpl

```yaml
{{/* database-app/templates/_helpers.tpl */}}

{{/*
Expand the name of the chart.
*/}}
{{- define "database-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "database-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "database-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "database-app.labels" -}}
helm.sh/chart: {{ include "database-app.chart" . }}
{{ include "database-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "database-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "database-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "database-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "database-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}

{{/*
PostgreSQL hostname
*/}}
{{- define "database-app.postgresql.primary" -}}
{{- if .Values.postgresql.enabled }}
{{- printf "%s-postgresql-primary" .Release.Name }}
{{- else }}
{{- .Values.externalPostgresql.host }}
{{- end }}
{{- end }}

{{/*
PostgreSQL readonly hostname
*/}}
{{- define "database-app.postgresql.read" -}}
{{- if .Values.postgresql.enabled }}
{{- printf "%s-postgresql-read" .Release.Name }}
{{- else }}
{{- .Values.externalPostgresql.readHost | default .Values.externalPostgresql.host }}
{{- end }}
{{- end }}

{{/*
Redis hostname
*/}}
{{- define "database-app.redis.host" -}}
{{- if .Values.redis.enabled }}
{{- printf "%s-redis-master" .Release.Name }}
{{- else }}
{{- .Values.externalRedis.host }}
{{- end }}
{{- end }}

{{/*
MinIO endpoint
*/}}
{{- define "database-app.minio.endpoint" -}}
{{- if .Values.minio.enabled }}
{{- printf "%s-minio:9000" .Release.Name }}
{{- else }}
{{- .Values.externalMinio.endpoint }}
{{- end }}
{{- end }}

{{/*
Database URL
*/}}
{{- define "database-app.databaseUrl" -}}
{{- printf "postgresql://%s:$(DB_PASSWORD)@%s:5432/%s" 
    .Values.postgresql.auth.username 
    (include "database-app.postgresql.primary" .)
    .Values.postgresql.auth.database }}
{{- end }}

{{/*
Generate random password if not set
*/}}
{{- define "database-app.randomPassword" -}}
{{- randAlphaNum 24 }}
{{- end }}
```

### 5.5 templates/deployment.yaml

```yaml
{{/* database-app/templates/deployment.yaml */}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "database-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "database-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.app.hpa.enabled }}
  replicas: {{ .Values.app.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "database-app.selectorLabels" . | nindent 6 }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        {{- include "database-app.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
        {{- if .Values.monitoring.enabled }}
        prometheus.io/scrape: "true"
        prometheus.io/port: {{ .Values.app.service.port | quote }}
        prometheus.io/path: "/metrics"
        {{- end }}
    spec:
      {{- with .Values.global.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.app.image.repository }}:{{ .Values.app.image.tag }}"
        imagePullPolicy: {{ .Values.app.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.app.service.port }}
          protocol: TCP
        env:
        # Static environment variables
        {{- range $key, $value := .Values.app.env }}
        - name: {{ $key }}
          value: {{ $value | quote }}
        {{- end }}
        # Database connection
        - name: DB_HOST
          value: {{ include "database-app.postgresql.primary" . | quote }}
        - name: DB_READ_HOST
          value: {{ include "database-app.postgresql.read" . | quote }}
        - name: DB_PORT
          value: "5432"
        - name: DB_NAME
          value: {{ .Values.postgresql.auth.database | quote }}
        - name: DB_USER
          value: {{ .Values.postgresql.auth.username | quote }}
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ include "database-app.fullname" . }}-secrets
              key: postgresql-password
        # Redis connection
        - name: REDIS_HOST
          value: {{ include "database-app.redis.host" . | quote }}
        - name: REDIS_PORT
          value: "6379"
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ include "database-app.fullname" . }}-secrets
              key: redis-password
        # MinIO connection
        - name: MINIO_ENDPOINT
          value: {{ include "database-app.minio.endpoint" . | quote }}
        - name: MINIO_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: {{ include "database-app.fullname" . }}-secrets
              key: minio-access-key
        - name: MINIO_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: {{ include "database-app.fullname" . }}-secrets
              key: minio-secret-key
        # Pod info
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        resources:
          {{- toYaml .Values.app.resources | nindent 10 }}
        livenessProbe:
          httpGet:
            path: /health/live
            port: http
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
      terminationGracePeriodSeconds: 30
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app.kubernetes.io/name
                  operator: In
                  values:
                  - {{ include "database-app.name" . }}
              topologyKey: kubernetes.io/hostname
```

### 5.6 templates/secret.yaml

```yaml
{{/* database-app/templates/secret.yaml */}}
{{- if .Values.secrets.create }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "database-app.fullname" . }}-secrets
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "database-app.labels" . | nindent 4 }}
  annotations:
    "helm.sh/resource-policy": keep
type: Opaque
data:
  postgresql-password: {{ .Values.postgresql.auth.password | default (randAlphaNum 24) | b64enc | quote }}
  redis-password: {{ .Values.redis.auth.password | default (randAlphaNum 24) | b64enc | quote }}
  minio-access-key: {{ .Values.minio.auth.rootUser | b64enc | quote }}
  minio-secret-key: {{ .Values.minio.auth.rootPassword | default (randAlphaNum 32) | b64enc | quote }}
{{- end }}
```

### 5.7 templates/hpa.yaml

```yaml
{{/* database-app/templates/hpa.yaml */}}
{{- if .Values.app.hpa.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "database-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "database-app.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "database-app.fullname" . }}
  minReplicas: {{ .Values.app.hpa.minReplicas }}
  maxReplicas: {{ .Values.app.hpa.maxReplicas }}
  metrics:
  {{- if .Values.app.hpa.targetCPUUtilizationPercentage }}
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: {{ .Values.app.hpa.targetCPUUtilizationPercentage }}
  {{- end }}
  {{- if .Values.app.hpa.targetMemoryUtilizationPercentage }}
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: {{ .Values.app.hpa.targetMemoryUtilizationPercentage }}
  {{- end }}
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
{{- end }}
```

### 5.8 templates/ingress.yaml

```yaml
{{/* database-app/templates/ingress.yaml */}}
{{- if .Values.app.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "database-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "database-app.labels" . | nindent 4 }}
  {{- with .Values.app.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.app.ingress.className }}
  ingressClassName: {{ .Values.app.ingress.className }}
  {{- end }}
  {{- if .Values.app.ingress.tls }}
  tls:
    {{- range .Values.app.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.app.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "database-app.fullname" $ }}
                port:
                  number: {{ $.Values.app.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```

### 5.9 templates/NOTES.txt

```
{{/* database-app/templates/NOTES.txt */}}
╔══════════════════════════════════════════════════════════╗
║          Database Application ติดตั้งสำเร็จ!            ║
╚══════════════════════════════════════════════════════════╝

Release name: {{ .Release.Name }}
Namespace:    {{ .Release.Namespace }}
Chart:        {{ .Chart.Name }}-{{ .Chart.Version }}
App Version:  {{ .Chart.AppVersion }}

═══════════════════════════════════════
🌐 Application Access:
═══════════════════════════════════════
{{- if .Values.app.ingress.enabled }}
{{- range .Values.app.ingress.hosts }}
  URL: https://{{ .host }}
{{- end }}
{{- else }}
  kubectl port-forward service/{{ include "database-app.fullname" . }} 3000:{{ .Values.app.service.port }} -n {{ .Release.Namespace }}
  จากนั้นเปิด: http://localhost:3000
{{- end }}

═══════════════════════════════════════
🗄️ Database Access:
═══════════════════════════════════════
{{- if .Values.postgresql.enabled }}
PostgreSQL Primary:
  kubectl port-forward service/{{ .Release.Name }}-postgresql-primary 5432:5432 -n {{ .Release.Namespace }}
  Host: {{ .Release.Name }}-postgresql-primary
  Port: 5432
  Database: {{ .Values.postgresql.auth.database }}
  User: {{ .Values.postgresql.auth.username }}
  
  รับ password:
  export POSTGRES_PASSWORD=$(kubectl get secret --namespace {{ .Release.Namespace }} {{ .Release.Name }}-postgresql -o jsonpath="{.data.password}" | base64 -d)
{{- end }}

{{- if .Values.redis.enabled }}
Redis:
  kubectl port-forward service/{{ .Release.Name }}-redis-master 6379:6379 -n {{ .Release.Namespace }}
  
  รับ password:
  export REDIS_PASSWORD=$(kubectl get secret --namespace {{ .Release.Namespace }} {{ .Release.Name }}-redis -o jsonpath="{.data.redis-password}" | base64 -d)
{{- end }}

═══════════════════════════════════════
📊 Monitoring:
═══════════════════════════════════════
  kubectl top pods -n {{ .Release.Namespace }}
  kubectl get pods -n {{ .Release.Namespace }} -w

═══════════════════════════════════════
⚠️ หมายเหตุ:
═══════════════════════════════════════
  - ตรวจสอบให้แน่ใจว่า StorageClass "{{ .Values.global.storageClass | default "fast-ssd" }}" มีอยู่ใน cluster
  - ตรวจสอบ resource quotas ใน namespace
  - ดู release notes ด้วย: helm get notes {{ .Release.Name }} -n {{ .Release.Namespace }}
```

---

## 6. Helm Hooks

### 6.1 Pre-install Hook

```yaml
# templates/hooks/pre-install-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "database-app.fullname" . }}-db-init
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "database-app.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 3
  template:
    metadata:
      labels:
        {{- include "database-app.selectorLabels" . | nindent 8 }}
    spec:
      restartPolicy: OnFailure
      initContainers:
      - name: wait-for-postgresql
        image: busybox:1.35
        command:
        - sh
        - -c
        - |
          until nc -z {{ include "database-app.postgresql.primary" . }} 5432; do
            echo "Waiting for PostgreSQL..."
            sleep 5
          done
          echo "PostgreSQL is ready!"
      containers:
      - name: db-migrate
        image: "{{ .Values.app.image.repository }}:{{ .Values.app.image.tag }}"
        command: ["node", "scripts/migrate.js"]
        env:
        - name: DATABASE_URL
          value: {{ include "database-app.databaseUrl" . | quote }}
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ include "database-app.fullname" . }}-secrets
              key: postgresql-password
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

### 6.2 Post-install Hook

```yaml
# templates/hooks/post-install-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "database-app.fullname" . }}-seed
  namespace: {{ .Release.Namespace }}
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "1"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: seed-data
        image: "{{ .Values.app.image.repository }}:{{ .Values.app.image.tag }}"
        command: ["node", "scripts/seed.js"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: {{ include "database-app.fullname" . }}-secrets
              key: postgresql-password
```

---

## 7. Helm Tests

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "database-app.fullname" . }}-test
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "database-app.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  restartPolicy: Never
  containers:
  - name: test-postgresql
    image: postgres:15
    command:
    - sh
    - -c
    - |
      # Test PostgreSQL connection
      PGPASSWORD=$DB_PASSWORD psql -h $DB_HOST -U $DB_USER -d $DB_NAME -c "SELECT version();"
      
      # Test basic query
      PGPASSWORD=$DB_PASSWORD psql -h $DB_HOST -U $DB_USER -d $DB_NAME -c "SELECT COUNT(*) FROM information_schema.tables;"
      
      echo "PostgreSQL test PASSED!"
    env:
    - name: DB_HOST
      value: {{ include "database-app.postgresql.primary" . | quote }}
    - name: DB_USER
      value: {{ .Values.postgresql.auth.username | quote }}
    - name: DB_NAME
      value: {{ .Values.postgresql.auth.database | quote }}
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: {{ include "database-app.fullname" . }}-secrets
          key: postgresql-password
          
  - name: test-redis
    image: redis:7
    command:
    - sh
    - -c
    - |
      # Test Redis connection
      redis-cli -h $REDIS_HOST -a $REDIS_PASSWORD ping
      
      # Test set/get
      redis-cli -h $REDIS_HOST -a $REDIS_PASSWORD SET test_key "hello_from_helm_test"
      VALUE=$(redis-cli -h $REDIS_HOST -a $REDIS_PASSWORD GET test_key)
      if [ "$VALUE" = "hello_from_helm_test" ]; then
        echo "Redis test PASSED!"
        redis-cli -h $REDIS_HOST -a $REDIS_PASSWORD DEL test_key
      else
        echo "Redis test FAILED!"
        exit 1
      fi
    env:
    - name: REDIS_HOST
      value: {{ include "database-app.redis.host" . | quote }}
    - name: REDIS_PASSWORD
      valueFrom:
        secretKeyRef:
          name: {{ include "database-app.fullname" . }}-secrets
          key: redis-password
```

```bash
# Run tests
helm test database-app -n applications

# ดู test output
kubectl logs database-app-test -n applications
```

---

## 8. Helmfile: จัดการหลาย Charts

### 8.1 helmfile.yaml

```yaml
# helmfile.yaml
environments:
  dev:
    values:
    - environments/dev/values.yaml
  staging:
    values:
    - environments/staging/values.yaml
  production:
    values:
    - environments/production/values.yaml

repositories:
- name: bitnami
  url: https://charts.bitnami.com/bitnami
- name: ingress-nginx
  url: https://kubernetes.github.io/ingress-nginx
- name: cert-manager
  url: https://charts.jetstack.io
- name: prometheus-community
  url: https://prometheus-community.github.io/helm-charts

releases:
# Infrastructure
- name: cert-manager
  namespace: cert-manager
  chart: cert-manager/cert-manager
  version: v1.13.0
  createNamespace: true
  values:
  - installCRDs: true
  wait: true

- name: ingress-nginx
  namespace: ingress-nginx
  chart: ingress-nginx/ingress-nginx
  version: 4.8.3
  createNamespace: true
  values:
  - controller:
      replicaCount: 2
      resources:
        requests:
          memory: "256Mi"
          cpu: "100m"
        limits:
          memory: "1Gi"
          cpu: "500m"
  wait: true
  needs:
  - cert-manager/cert-manager

# Monitoring
- name: prometheus-stack
  namespace: monitoring
  chart: prometheus-community/kube-prometheus-stack
  version: 55.5.0
  createNamespace: true
  values:
  - environments/{{ .Environment.Name }}/prometheus-values.yaml
  wait: true

# Databases
- name: postgresql
  namespace: databases
  chart: bitnami/postgresql
  version: 13.4.0
  createNamespace: true
  values:
  - environments/{{ .Environment.Name }}/postgresql-values.yaml
  wait: true
  needs:
  - monitoring/prometheus-stack

- name: redis
  namespace: databases
  chart: bitnami/redis
  version: 18.6.0
  values:
  - environments/{{ .Environment.Name }}/redis-values.yaml
  wait: true
  needs:
  - monitoring/prometheus-stack

# Application
- name: database-app
  namespace: applications
  chart: ./charts/database-app
  version: 1.0.0
  createNamespace: true
  values:
  - environments/{{ .Environment.Name }}/app-values.yaml
  wait: true
  needs:
  - databases/postgresql
  - databases/redis
```

```bash
# Install ทั้งหมด
helmfile apply

# Install เฉพาะบาง environments
helmfile -e production apply

# Diff ก่อน apply
helmfile diff

# Sync specific release
helmfile -l name=database-app sync

# Destroy ทั้งหมด
helmfile destroy
```

---

## 9. Environment-specific Values

### 9.1 Dev Values

```yaml
# environments/dev/app-values.yaml
app:
  replicaCount: 1
  image:
    tag: "latest"
  resources:
    requests:
      memory: "128Mi"
      cpu: "50m"
    limits:
      memory: "512Mi"
      cpu: "200m"
  hpa:
    enabled: false
  pdb:
    enabled: false
  ingress:
    enabled: true
    hosts:
    - host: api.dev.example.com
      paths:
      - path: /
        pathType: Prefix
    tls: []

postgresql:
  primary:
    persistence:
      size: 5Gi
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "1Gi"
        cpu: "500m"
  readReplicas:
    replicaCount: 1
    persistence:
      size: 5Gi

redis:
  master:
    persistence:
      size: 1Gi
  replica:
    replicaCount: 1
```

### 9.2 Production Values

```yaml
# environments/production/app-values.yaml
app:
  replicaCount: 5
  image:
    tag: "1.0.0"
    pullPolicy: IfNotPresent
  resources:
    requests:
      memory: "512Mi"
      cpu: "250m"
    limits:
      memory: "2Gi"
      cpu: "1000m"
  hpa:
    enabled: true
    minReplicas: 5
    maxReplicas: 50
    targetCPUUtilizationPercentage: 70
  pdb:
    enabled: true
    minAvailable: 3
  ingress:
    enabled: true
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
      nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    hosts:
    - host: api.example.com
      paths:
      - path: /
        pathType: Prefix
    tls:
    - secretName: api-tls
      hosts:
      - api.example.com

postgresql:
  primary:
    persistence:
      size: 100Gi
      storageClass: fast-ssd
    resources:
      requests:
        memory: "2Gi"
        cpu: "1000m"
      limits:
        memory: "8Gi"
        cpu: "4000m"
  readReplicas:
    replicaCount: 3
    persistence:
      size: 100Gi

redis:
  master:
    persistence:
      size: 20Gi
  replica:
    replicaCount: 3
```

---

## 10. Helm Secrets Plugin

```bash
# ติดตั้ง helm-secrets plugin
helm plugin install https://github.com/jkroepke/helm-secrets

# ติดตั้ง SOPS (Secret OPerationS)
brew install sops

# สร้าง GPG key หรือใช้ AWS KMS
# สร้าง secrets ที่ encrypted
sops --encrypt --in-place secrets.yaml

# ดู secrets file ที่ encrypted
cat secrets.yaml

# Install ด้วย encrypted secrets
helm secrets install myapp ./charts/myapp \
  -f values.yaml \
  -f secrets.yaml \
  -n production

# Decrypt สำหรับ debug
helm secrets dec secrets.yaml
```

---

## 11. สรุป

Helm ช่วยลดความซับซ้อนในการ manage Kubernetes applications:

1. **Charts** รวม manifests ทั้งหมดไว้ในที่เดียว
2. **Values** ช่วย customize ตาม environment
3. **Templates** ลดการ duplicate code
4. **Hooks** จัดการ lifecycle events
5. **Tests** validate การ deploy
6. **Helmfile** จัดการหลาย charts พร้อมกัน

### Commands สรุป

```bash
# ติดตั้ง
helm install myapp ./charts/database-app -n production -f values-prod.yaml

# Upgrade
helm upgrade myapp ./charts/database-app -n production -f values-prod.yaml --wait

# Rollback
helm rollback myapp 1 -n production

# Test
helm test myapp -n production

# List
helm list -n production

# Uninstall
helm uninstall myapp -n production
```
