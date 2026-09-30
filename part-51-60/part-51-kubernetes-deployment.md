# Part 51: Kubernetes Deployment สำหรับ Database Cluster

## บทนำ

Kubernetes (K8s) เป็น container orchestration platform ที่ทรงพลังที่สุดในปัจจุบัน สำหรับการ deploy database cluster บน Kubernetes นั้น มีความท้าทายและข้อควรพิจารณาหลายประการที่แตกต่างจาก stateless applications ทั่วไป บทนี้จะครอบคลุมทุกแง่มุมของการ deploy PostgreSQL, Redis และ MinIO บน Kubernetes อย่างครบถ้วน

---

## 1. Kubernetes Concepts สำหรับ Database Deployments

### 1.1 ทำไม Kubernetes ถึงยากสำหรับ Databases

Database workloads มีความพิเศษหลายประการ:

```
Stateless Apps          vs      Stateful Apps (Databases)
─────────────────────────────────────────────────────────
- ไม่มี state                   - มี persistent data
- สามารถ scale แบบ random       - ต้อง scale อย่างมีระเบียบ
- Pod ใดก็เหมือนกัน             - แต่ละ Pod มี identity ต่างกัน
- ลบแล้วสร้างใหม่ได้            - ต้องรักษา data เดิม
- ไม่ต้องการ DNS ตายตัว         - ต้องการ stable network identity
```

### 1.2 Kubernetes Objects ที่สำคัญสำหรับ Databases

#### Pod
หน่วยการ deploy ที่เล็กที่สุดใน Kubernetes

```yaml
# ตัวอย่าง Pod manifest พื้นฐาน (ไม่แนะนำสำหรับ production)
apiVersion: v1
kind: Pod
metadata:
  name: postgresql-pod
  labels:
    app: postgresql
spec:
  containers:
  - name: postgresql
    image: postgres:15
    env:
    - name: POSTGRES_PASSWORD
      value: "mysecretpassword"
    ports:
    - containerPort: 5432
```

#### Deployment
จัดการ stateless applications

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
```

#### StatefulSet
จัดการ stateful applications เช่น databases

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
spec:
  serviceName: postgresql-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgresql
  template:
    # ...
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

#### Service Types

```
ClusterIP     - เข้าถึงได้ภายใน cluster เท่านั้น
NodePort      - เปิด port บน Node IP
LoadBalancer  - สร้าง external load balancer
Headless      - ไม่มี cluster IP, ให้ DNS ตรงไปยัง Pod
ExternalName  - map ไปยัง DNS ภายนอก
```

#### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgresql-config
data:
  postgresql.conf: |
    max_connections = 200
    shared_buffers = 256MB
    effective_cache_size = 1GB
    work_mem = 4MB
    maintenance_work_mem = 64MB
    wal_level = replica
    max_wal_senders = 10
    wal_keep_size = 1GB
  pg_hba.conf: |
    local   all             all                                     trust
    host    all             all             127.0.0.1/32            md5
    host    all             all             ::1/128                 md5
    host    all             all             10.0.0.0/8              md5
    host    replication     replicator      10.0.0.0/8              md5
```

#### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgresql-secret
type: Opaque
data:
  # base64 encoded values
  postgres-password: bXlzdXBlcnNlY3JldHBhc3N3b3Jk
  replication-password: cmVwbGljYXRpb25zZWNyZXQ=
```

การสร้าง Secret จาก command line:
```bash
# สร้าง base64 encoded value
echo -n "mysupersecretpassword" | base64

# สร้าง Secret ด้วย kubectl
kubectl create secret generic postgresql-secret \
  --from-literal=postgres-password=mysupersecretpassword \
  --from-literal=replication-password=replicationsecret \
  -n production
```

#### PersistentVolume และ PersistentVolumeClaim

```yaml
# PersistentVolume (PV) - กำหนดโดย admin
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgresql-pv-0
spec:
  capacity:
    storage: 50Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast-ssd
  hostPath:
    path: /mnt/data/postgresql-0

---
# PersistentVolumeClaim (PVC) - ร้องขอโดย user
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql-data-postgresql-0
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 50Gi
```

#### StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

---

## 2. StatefulSet: ทำไมต้องใช้สำหรับ Databases

### 2.1 Stable Network Identity

StatefulSet ให้ Pod ชื่อที่คาดเดาได้และ stable:

```
Deployment Pod names:    myapp-7d4b9c8f7-xk2j9   (random)
StatefulSet Pod names:   postgresql-0              (ตายตัว)
                         postgresql-1
                         postgresql-2
```

DNS records ที่ StatefulSet สร้าง:
```
postgresql-0.postgresql-headless.default.svc.cluster.local
postgresql-1.postgresql-headless.default.svc.cluster.local
postgresql-2.postgresql-headless.default.svc.cluster.local
```

### 2.2 Ordered Deployment และ Scaling

```
StatefulSet เริ่ม Pod ตามลำดับ:
postgresql-0  → Running  →  postgresql-1  → Running  →  postgresql-2

ลด replicas ตามลำดับย้อนกลับ:
postgresql-2  → Terminated  →  postgresql-1  → Terminated

ทำให้ผู้ดูแลระบบสามารถ:
1. สร้าง primary node ก่อน (postgresql-0)
2. สร้าง replica nodes ตามลำดับ
3. ลบ replicas อย่างปลอดภัยจากตัวล่างสุด
```

### 2.3 Persistent Storage

```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: ["ReadWriteOnce"]
    storageClassName: fast-ssd
    resources:
      requests:
        storage: 50Gi
```

แต่ละ Pod จะได้ PVC ของตัวเอง:
```
postgresql-0  →  data-postgresql-0  (50Gi)
postgresql-1  →  data-postgresql-1  (50Gi)
postgresql-2  →  data-postgresql-2  (50Gi)
```

เมื่อ Pod ถูกลบแล้วสร้างใหม่ จะ attach กลับไปที่ PVC เดิม

---

## 3. PostgreSQL บน Kubernetes

### 3.1 Namespace Setup

```bash
# สร้าง namespace
kubectl create namespace databases
kubectl create namespace applications

# Label namespace สำหรับ network policies
kubectl label namespace databases purpose=database
kubectl label namespace applications purpose=application
```

### 3.2 PostgreSQL Secret

```yaml
# postgresql-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgresql-secret
  namespace: databases
type: Opaque
stringData:
  postgres-password: "P@ssw0rd#2024"
  replication-password: "Repl!cat!on#2024"
  app-password: "App#P@ss2024"
```

```bash
kubectl apply -f postgresql-secret.yaml
```

### 3.3 PostgreSQL ConfigMap

```yaml
# postgresql-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgresql-config
  namespace: databases
data:
  postgresql.conf: |
    # Connection Settings
    listen_addresses = '*'
    max_connections = 300
    superuser_reserved_connections = 10
    
    # Memory Settings
    shared_buffers = 512MB
    effective_cache_size = 2GB
    work_mem = 8MB
    maintenance_work_mem = 128MB
    huge_pages = off
    
    # WAL Settings
    wal_level = replica
    max_wal_senders = 10
    max_replication_slots = 10
    wal_keep_size = 2GB
    wal_compression = on
    
    # Replication Settings
    hot_standby = on
    hot_standby_feedback = on
    
    # Checkpoint Settings
    checkpoint_timeout = 15min
    checkpoint_completion_target = 0.9
    max_wal_size = 4GB
    min_wal_size = 1GB
    
    # Query Planner
    random_page_cost = 1.1
    effective_io_concurrency = 200
    
    # Logging
    log_timezone = 'Asia/Bangkok'
    log_destination = 'stderr'
    logging_collector = off
    log_min_duration_statement = 1000
    log_checkpoints = on
    log_connections = on
    log_disconnections = on
    log_lock_waits = on
    log_temp_files = 0
    
    # Statistics
    track_activities = on
    track_counts = on
    track_io_timing = on
    
    # Locale
    datestyle = 'iso, mdy'
    timezone = 'Asia/Bangkok'
    lc_messages = 'en_US.utf8'
    lc_monetary = 'en_US.utf8'
    lc_numeric = 'en_US.utf8'
    lc_time = 'en_US.utf8'
    default_text_search_config = 'pg_catalog.english'
    
  pg_hba.conf: |
    # TYPE  DATABASE        USER            ADDRESS                 METHOD
    local   all             all                                     trust
    host    all             all             127.0.0.1/32            md5
    host    all             all             ::1/128                 md5
    # อนุญาต connections จาก Kubernetes pods
    host    all             all             10.0.0.0/8              md5
    host    all             all             172.16.0.0/12           md5
    host    all             all             192.168.0.0/16          md5
    # Replication
    host    replication     replicator      10.0.0.0/8              md5
    host    replication     replicator      172.16.0.0/12           md5
    
  init-db.sql: |
    -- สร้าง user สำหรับ application
    CREATE USER appuser WITH PASSWORD 'App#P@ss2024';
    
    -- สร้าง database
    CREATE DATABASE appdb OWNER appuser;
    
    -- สร้าง replication user
    CREATE USER replicator WITH REPLICATION PASSWORD 'Repl!cat!on#2024';
    
    -- Grant permissions
    GRANT ALL PRIVILEGES ON DATABASE appdb TO appuser;
    
    \c appdb
    CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
    CREATE EXTENSION IF NOT EXISTS pgcrypto;
    CREATE EXTENSION IF NOT EXISTS uuid-ossp;
```

### 3.4 PostgreSQL StatefulSet

```yaml
# postgresql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: databases
  labels:
    app: postgresql
    version: "15"
spec:
  serviceName: postgresql-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
        version: "15"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9187"
    spec:
      # Security context สำหรับ volume permissions
      securityContext:
        fsGroup: 999
        runAsUser: 999
        runAsNonRoot: true
        
      # Init container สำหรับ setup permissions
      initContainers:
      - name: init-permissions
        image: busybox:1.35
        command:
        - sh
        - -c
        - |
          mkdir -p /data/pgdata
          chown -R 999:999 /data
          chmod 700 /data/pgdata
        volumeMounts:
        - name: data
          mountPath: /data
        securityContext:
          runAsUser: 0
          
      - name: init-config
        image: postgres:15
        command:
        - sh
        - -c
        - |
          # กำหนด primary/replica ตาม ordinal index
          POD_INDEX=${POD_NAME##*-}
          echo "Pod index: $POD_INDEX"
          
          if [ "$POD_INDEX" = "0" ]; then
            echo "This is PRIMARY node"
            echo "primary" > /shared/role
          else
            echo "This is REPLICA node $POD_INDEX"
            echo "replica" > /shared/role
            echo "$POSTGRES_MASTER_HOST" > /shared/master
          fi
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POSTGRES_MASTER_HOST
          value: "postgresql-0.postgresql-headless.databases.svc.cluster.local"
        volumeMounts:
        - name: shared-data
          mountPath: /shared
          
      containers:
      # Main PostgreSQL container
      - name: postgresql
        image: postgres:15
        imagePullPolicy: IfNotPresent
        ports:
        - name: postgresql
          containerPort: 5432
          protocol: TCP
          
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgresql-secret
              key: postgres-password
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_DB
          value: postgres
        - name: PGDATA
          value: /data/pgdata
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
              
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
            
        # Volume mounts
        volumeMounts:
        - name: data
          mountPath: /data
        - name: config
          mountPath: /etc/postgresql/postgresql.conf
          subPath: postgresql.conf
        - name: config
          mountPath: /etc/postgresql/pg_hba.conf
          subPath: pg_hba.conf
        - name: init-scripts
          mountPath: /docker-entrypoint-initdb.d/
        - name: shared-data
          mountPath: /shared
          
        # Liveness probe
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - exec pg_isready -U postgres -h localhost -p 5432
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
          
        # Readiness probe
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - exec pg_isready -U postgres -h localhost -p 5432
          initialDelaySeconds: 30
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
          
        # Startup probe - รอ database เริ่มต้น
        startupProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - exec pg_isready -U postgres -h localhost -p 5432
          failureThreshold: 30
          periodSeconds: 10
          
      # PostgreSQL Exporter สำหรับ Prometheus
      - name: postgresql-exporter
        image: prometheuscommunity/postgres-exporter:latest
        ports:
        - name: metrics
          containerPort: 9187
        env:
        - name: DATA_SOURCE_NAME
          value: "postgresql://postgres:$(POSTGRES_PASSWORD)@localhost:5432/postgres?sslmode=disable"
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgresql-secret
              key: postgres-password
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "256Mi"
            cpu: "200m"
            
      # Volumes ที่ไม่ใช่ PVC
      volumes:
      - name: config
        configMap:
          name: postgresql-config
          items:
          - key: postgresql.conf
            path: postgresql.conf
          - key: pg_hba.conf
            path: pg_hba.conf
      - name: init-scripts
        configMap:
          name: postgresql-config
          items:
          - key: init-db.sql
            path: init-db.sql
      - name: shared-data
        emptyDir: {}
        
      # Anti-affinity: กระจาย Pods ไปยัง Nodes ต่าง ๆ
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - postgresql
            topologyKey: kubernetes.io/hostname
            
      # Tolerations
      tolerations:
      - key: "database"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
        
  # PVC templates
  volumeClaimTemplates:
  - metadata:
      name: data
      labels:
        app: postgresql
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 50Gi
```

### 3.5 PostgreSQL Services

```yaml
# postgresql-services.yaml
---
# Headless Service สำหรับ StatefulSet DNS
apiVersion: v1
kind: Service
metadata:
  name: postgresql-headless
  namespace: databases
  labels:
    app: postgresql
spec:
  type: ClusterIP
  clusterIP: None  # Headless
  publishNotReadyAddresses: false
  selector:
    app: postgresql
  ports:
  - name: postgresql
    port: 5432
    targetPort: postgresql
    protocol: TCP

---
# Service สำหรับ primary (write) connections
apiVersion: v1
kind: Service
metadata:
  name: postgresql-primary
  namespace: databases
  labels:
    app: postgresql
    role: primary
spec:
  type: ClusterIP
  selector:
    app: postgresql
    role: primary  # ต้องมี label นี้บน primary Pod
  ports:
  - name: postgresql
    port: 5432
    targetPort: postgresql
    protocol: TCP

---
# Service สำหรับ read-only connections (ทุก replica)
apiVersion: v1
kind: Service
metadata:
  name: postgresql-replicas
  namespace: databases
  labels:
    app: postgresql
    role: replica
spec:
  type: ClusterIP
  selector:
    app: postgresql
    role: replica
  ports:
  - name: postgresql
    port: 5432
    targetPort: postgresql
    protocol: TCP
```

---

## 4. Redis บน Kubernetes

### 4.1 Redis ConfigMap

```yaml
# redis-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: databases
data:
  redis.conf: |
    # Network
    bind 0.0.0.0
    protected-mode yes
    port 6379
    
    # General
    daemonize no
    supervised no
    pidfile /var/run/redis_6379.pid
    loglevel notice
    logfile ""
    databases 16
    
    # Security
    requirepass ${REDIS_PASSWORD}
    
    # Persistence
    save 900 1
    save 300 10
    save 60 10000
    
    stop-writes-on-bgsave-error yes
    rdbcompression yes
    rdbchecksum yes
    dbfilename dump.rdb
    dir /data
    
    # AOF
    appendonly yes
    appendfilename "appendonly.aof"
    appendfsync everysec
    no-appendfsync-on-rewrite no
    auto-aof-rewrite-percentage 100
    auto-aof-rewrite-min-size 64mb
    
    # Memory
    maxmemory 2gb
    maxmemory-policy allkeys-lru
    
    # Replication
    repl-diskless-sync no
    repl-diskless-sync-delay 5
    repl-backlog-size 1mb
    
    # Slow log
    slowlog-log-slower-than 10000
    slowlog-max-len 128
    
    # Latency
    latency-monitor-threshold 100
    
    # Advanced config
    hash-max-listpack-entries 128
    hash-max-listpack-value 64
    list-max-listpack-size -2
    list-compress-depth 0
    set-max-intset-entries 512
    zset-max-listpack-entries 128
    zset-max-listpack-value 64
    activerehashing yes
    hz 10
    aof-rewrite-incremental-fsync yes
    rdb-save-incremental-fsync yes
```

### 4.2 Redis Secret

```yaml
# redis-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: redis-secret
  namespace: databases
type: Opaque
stringData:
  redis-password: "Red!s#P@ss2024"
```

### 4.3 Redis StatefulSet

```yaml
# redis-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: databases
  labels:
    app: redis
spec:
  serviceName: redis-headless
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9121"
    spec:
      securityContext:
        fsGroup: 1000
        runAsUser: 1000
        runAsNonRoot: true
        
      initContainers:
      - name: init-redis
        image: redis:7
        command:
        - sh
        - -c
        - |
          set -ex
          POD_INDEX=${HOSTNAME##*-}
          
          # สร้าง redis.conf จาก template
          cp /config/redis.conf /data/redis.conf
          
          # กำหนดค่า replication
          if [ "$POD_INDEX" != "0" ]; then
            echo "replicaof redis-0.redis-headless.databases.svc.cluster.local 6379" >> /data/redis.conf
            echo "masterauth ${REDIS_PASSWORD}" >> /data/redis.conf
          fi
          
          # ใส่ password
          sed -i "s/\${REDIS_PASSWORD}/${REDIS_PASSWORD}/g" /data/redis.conf
          
          echo "Redis configuration for pod $HOSTNAME:"
          cat /data/redis.conf | grep -E "^(requirepass|replicaof|masterauth|appendonly|maxmemory)"
        env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: redis-password
        volumeMounts:
        - name: config
          mountPath: /config
        - name: data
          mountPath: /data
        securityContext:
          runAsUser: 0
          
      containers:
      - name: redis
        image: redis:7
        command:
        - redis-server
        - /data/redis.conf
        ports:
        - name: redis
          containerPort: 6379
        env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: redis-password
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "4Gi"
            cpu: "1000m"
        volumeMounts:
        - name: data
          mountPath: /data
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - redis-cli -a $REDIS_PASSWORD ping | grep PONG
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - redis-cli -a $REDIS_PASSWORD ping | grep PONG
          initialDelaySeconds: 15
          periodSeconds: 5
          
      # Redis Exporter สำหรับ Prometheus
      - name: redis-exporter
        image: oliver006/redis_exporter:latest
        ports:
        - name: metrics
          containerPort: 9121
        env:
        - name: REDIS_ADDR
          value: "redis://localhost:6379"
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: redis-password
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "256Mi"
            cpu: "200m"
            
      volumes:
      - name: config
        configMap:
          name: redis-config
          
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: redis
              topologyKey: kubernetes.io/hostname
              
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 10Gi
```

### 4.4 Redis Services

```yaml
# redis-services.yaml
---
# Headless Service
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
  namespace: databases
  labels:
    app: redis
spec:
  type: ClusterIP
  clusterIP: None
  selector:
    app: redis
  ports:
  - name: redis
    port: 6379
    targetPort: redis

---
# Service สำหรับ writes (master only)
apiVersion: v1
kind: Service
metadata:
  name: redis-master
  namespace: databases
  labels:
    app: redis
    role: master
spec:
  type: ClusterIP
  selector:
    app: redis
    statefulset.kubernetes.io/pod-name: redis-0
  ports:
  - name: redis
    port: 6379
    targetPort: redis

---
# Service สำหรับ reads (all replicas)
apiVersion: v1
kind: Service
metadata:
  name: redis-replicas
  namespace: databases
  labels:
    app: redis
spec:
  type: ClusterIP
  selector:
    app: redis
  ports:
  - name: redis
    port: 6379
    targetPort: redis
```

---

## 5. MinIO บน Kubernetes

### 5.1 MinIO Operator (แนะนำสำหรับ Production)

```bash
# ติดตั้ง MinIO Operator
kubectl apply -f https://raw.githubusercontent.com/minio/operator/master/docs/minio-operator.yaml

# ตรวจสอบ
kubectl get pods -n minio-operator
```

### 5.2 MinIO Tenant

```yaml
# minio-tenant.yaml
apiVersion: minio.min.io/v2
kind: Tenant
metadata:
  name: minio-tenant
  namespace: minio
spec:
  # จำนวน MinIO servers
  servers: 4
  
  # จำนวน drives ต่อ server
  volumesPerServer: 2
  
  # Storage class
  storageClassName: fast-ssd
  
  # Storage size ต่อ volume
  volumeSize: 100Gi
  
  # Image
  image: minio/minio:latest
  
  # Credentials
  credsSecret:
    name: minio-secret
    
  # Resource requests
  resources:
    requests:
      memory: "2Gi"
      cpu: "1000m"
    limits:
      memory: "8Gi"
      cpu: "4000m"
      
  # Features
  features:
    bucketDNS: true
    domains: {}
    
  # Console
  console:
    resources:
      requests:
        memory: "512Mi"
        cpu: "250m"
```

### 5.3 MinIO แบบ Simple StatefulSet

```yaml
# minio-statefulset.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: minio-config
  namespace: databases
data:
  MINIO_SITE_NAME: "k8s-minio"
  MINIO_SITE_REGION: "th-central-1"

---
apiVersion: v1
kind: Secret
metadata:
  name: minio-secret
  namespace: databases
type: Opaque
stringData:
  rootUser: "minioadmin"
  rootPassword: "Min!o#P@ss2024"

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: minio
  namespace: databases
  labels:
    app: minio
spec:
  serviceName: minio-headless
  replicas: 4
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio:latest
        command:
        - /bin/bash
        - -c
        - |
          # สร้าง argument list สำหรับ distributed mode
          PODS=""
          for i in $(seq 0 3); do
            PODS="$PODS http://minio-${i}.minio-headless.databases.svc.cluster.local:9000/data{1...2}"
          done
          exec minio server $PODS --console-address :9001
        ports:
        - name: minio
          containerPort: 9000
        - name: console
          containerPort: 9001
        env:
        - name: MINIO_ROOT_USER
          valueFrom:
            secretKeyRef:
              name: minio-secret
              key: rootUser
        - name: MINIO_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: minio-secret
              key: rootPassword
        - name: MINIO_DISTRIBUTED_MODE_ENABLED
          value: "yes"
        envFrom:
        - configMapRef:
            name: minio-config
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "8Gi"
            cpu: "4000m"
        volumeMounts:
        - name: data-1
          mountPath: /data1
        - name: data-2
          mountPath: /data2
        livenessProbe:
          httpGet:
            path: /minio/health/live
            port: 9000
          initialDelaySeconds: 30
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /minio/health/ready
            port: 9000
          initialDelaySeconds: 15
          periodSeconds: 15
          
  volumeClaimTemplates:
  - metadata:
      name: data-1
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
  - metadata:
      name: data-2
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi

---
apiVersion: v1
kind: Service
metadata:
  name: minio-headless
  namespace: databases
spec:
  type: ClusterIP
  clusterIP: None
  selector:
    app: minio
  ports:
  - name: minio
    port: 9000
  - name: console
    port: 9001

---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: databases
spec:
  type: ClusterIP
  selector:
    app: minio
  ports:
  - name: minio
    port: 9000
    targetPort: 9000
  - name: console
    port: 9001
    targetPort: 9001
```

---

## 6. Application Deployment

### 6.1 Application Deployment Manifest

```yaml
# app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
  namespace: applications
  labels:
    app: nodejs-app
    version: "1.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nodejs-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: nodejs-app
        version: "1.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
        
      containers:
      - name: nodejs-app
        image: myregistry.io/nodejs-app:1.0
        imagePullPolicy: Always
        ports:
        - name: http
          containerPort: 3000
          
        env:
        # Database connection (Primary - สำหรับ writes)
        - name: DB_HOST
          value: "postgresql-primary.databases.svc.cluster.local"
        - name: DB_PORT
          value: "5432"
        - name: DB_NAME
          value: "appdb"
        - name: DB_USER
          value: "appuser"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db-password
              
        # Database readonly (สำหรับ reads)
        - name: DB_READ_HOST
          value: "postgresql-replicas.databases.svc.cluster.local"
          
        # Redis connection
        - name: REDIS_HOST
          value: "redis-master.databases.svc.cluster.local"
        - name: REDIS_PORT
          value: "6379"
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: redis-password
              
        # MinIO connection
        - name: MINIO_ENDPOINT
          value: "minio.databases.svc.cluster.local:9000"
        - name: MINIO_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: minio-access-key
        - name: MINIO_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: minio-secret-key
              
        # App settings
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "3000"
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
              
        # Resource limits
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "1Gi"
            cpu: "500m"
            
        # Health probes
        livenessProbe:
          httpGet:
            path: /health/live
            port: 3000
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
          
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
          
        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
              
      terminationGracePeriodSeconds: 30
      
      # Image pull secrets
      imagePullSecrets:
      - name: registry-secret
      
      # Anti-affinity
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: nodejs-app
              topologyKey: kubernetes.io/hostname
```

### 6.2 HorizontalPodAutoscaler

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nodejs-app-hpa
  namespace: applications
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nodejs-app
  minReplicas: 3
  maxReplicas: 20
  metrics:
  # CPU-based scaling
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # Memory-based scaling  
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # Custom metric: requests per second
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60
      - type: Percent
        value: 100
        periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
```

### 6.3 PodDisruptionBudget

```yaml
# pdb.yaml
---
# สำหรับ Application
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nodejs-app-pdb
  namespace: applications
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: nodejs-app

---
# สำหรับ PostgreSQL
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgresql-pdb
  namespace: databases
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: postgresql

---
# สำหรับ Redis
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: redis-pdb
  namespace: databases
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: redis
```

---

## 7. Namespaces และ Network Policies

### 7.1 Namespace Configuration

```yaml
# namespaces.yaml
---
# Development namespace
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    environment: dev
    team: backend

---
# Staging namespace
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
    team: backend

---
# Production namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: backend

---
# Databases namespace
apiVersion: v1
kind: Namespace
metadata:
  name: databases
  labels:
    purpose: database
```

### 7.2 NetworkPolicy

```yaml
# network-policies.yaml
---
# อนุญาตให้ application pods เข้าถึง PostgreSQL
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-postgresql
  namespace: databases
spec:
  podSelector:
    matchLabels:
      app: postgresql
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: application
    ports:
    - protocol: TCP
      port: 5432

---
# อนุญาตให้ application pods เข้าถึง Redis
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-redis
  namespace: databases
spec:
  podSelector:
    matchLabels:
      app: redis
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: application
    ports:
    - protocol: TCP
      port: 6379

---
# Default deny all ingress ใน databases namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: databases
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

### 7.3 ResourceQuota

```yaml
# resource-quota.yaml
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "50"
    services: "20"
    persistentvolumeclaims: "30"
    requests.storage: "500Gi"

---
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "4000m"
      memory: "8Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
  - type: Pod
    max:
      cpu: "8000m"
      memory: "16Gi"
```

---

## 8. RBAC Configuration

```yaml
# rbac.yaml
---
# ServiceAccount สำหรับ application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: applications

---
# Role: อนุญาตให้อ่าน secrets ใน namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: applications
rules:
- apiGroups: [""]
  resources: ["secrets", "configmaps"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-role-binding
  namespace: applications
subjects:
- kind: ServiceAccount
  name: app-service-account
  namespace: applications
roleRef:
  kind: Role
  name: app-role
  apiGroup: rbac.authorization.k8s.io
```

---

## 9. Ingress Configuration

```yaml
# ingress.yaml
---
# Ingress สำหรับ Application
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: applications
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nodejs-app
            port:
              number: 3000

---
# Ingress สำหรับ MinIO Console
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minio-console-ingress
  namespace: databases
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/proxy-body-size: "500m"
spec:
  rules:
  - host: minio.internal.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: minio
            port:
              number: 9001
```

---

## 10. kubectl Commands ที่ใช้บ่อย

### 10.1 การ Deploy

```bash
# Apply manifests
kubectl apply -f postgresql-secret.yaml
kubectl apply -f postgresql-config.yaml
kubectl apply -f postgresql-statefulset.yaml
kubectl apply -f postgresql-services.yaml

# Apply ทั้ง directory
kubectl apply -f ./k8s/ -n databases

# Apply ด้วย dry-run เพื่อ validate ก่อน
kubectl apply -f postgresql-statefulset.yaml --dry-run=client
kubectl apply -f postgresql-statefulset.yaml --dry-run=server

# ดู diff ก่อน apply
kubectl diff -f postgresql-statefulset.yaml
```

### 10.2 การ Monitor

```bash
# ดู pods ทั้งหมด
kubectl get pods -n databases
kubectl get pods -n databases -o wide   # เห็น Node ที่อยู่

# Watch pods แบบ real-time
kubectl get pods -n databases -w

# ดู pod details
kubectl describe pod postgresql-0 -n databases

# ดู logs
kubectl logs postgresql-0 -n databases
kubectl logs postgresql-0 -n databases -f          # follow
kubectl logs postgresql-0 -n databases --tail=100  # ล่าสุด 100 บรรทัด
kubectl logs postgresql-0 -n databases -c postgresql-exporter  # specific container

# ดู events
kubectl get events -n databases --sort-by=.lastTimestamp
kubectl get events -n databases --field-selector reason=Failed

# ดู StatefulSet
kubectl get statefulsets -n databases
kubectl describe statefulset postgresql -n databases

# ดู PVC
kubectl get pvc -n databases
kubectl describe pvc data-postgresql-0 -n databases

# ดู services
kubectl get services -n databases
kubectl describe service postgresql-primary -n databases
```

### 10.3 การ Debug

```bash
# เข้าไปใน container
kubectl exec -it postgresql-0 -n databases -- bash
kubectl exec -it postgresql-0 -n databases -c postgresql -- psql -U postgres

# Port forward เพื่อ test locally
kubectl port-forward pod/postgresql-0 5432:5432 -n databases
kubectl port-forward service/postgresql-primary 5432:5432 -n databases

# ดู resource usage
kubectl top pods -n databases
kubectl top nodes

# ดู node information
kubectl get nodes -o wide
kubectl describe node worker-01
```

### 10.4 การ Scale

```bash
# Scale statefulset
kubectl scale statefulset postgresql --replicas=5 -n databases

# Scale deployment
kubectl scale deployment nodejs-app --replicas=10 -n applications

# ดู rollout status
kubectl rollout status statefulset/postgresql -n databases
kubectl rollout status deployment/nodejs-app -n applications

# Rollback
kubectl rollout undo deployment/nodejs-app -n applications
kubectl rollout undo deployment/nodejs-app --to-revision=2 -n applications

# ดู rollout history
kubectl rollout history deployment/nodejs-app -n applications
```

### 10.5 การจัดการ Secrets

```bash
# ดู secrets (ไม่เห็น values)
kubectl get secrets -n databases
kubectl describe secret postgresql-secret -n databases

# Decode secret value
kubectl get secret postgresql-secret -n databases \
  -o jsonpath='{.data.postgres-password}' | base64 -d

# สร้าง secret จาก literal
kubectl create secret generic app-secrets \
  --from-literal=db-password="P@ssw0rd#2024" \
  --from-literal=redis-password="Red!s#P@ss2024" \
  -n applications

# สร้าง secret จาก file
kubectl create secret generic tls-secret \
  --from-file=tls.crt=./cert.pem \
  --from-file=tls.key=./key.pem \
  -n applications

# อัปเดต secret
kubectl create secret generic postgresql-secret \
  --from-literal=postgres-password="NewP@ssw0rd#2024" \
  -n databases \
  --dry-run=client -o yaml | kubectl apply -f -
```

### 10.6 การจัดการ Namespaces

```bash
# สร้าง namespace
kubectl create namespace staging

# ดู resources ทุก namespace
kubectl get pods --all-namespaces
kubectl get pods -A

# Set default namespace
kubectl config set-context --current --namespace=databases

# ดู current context
kubectl config current-context
kubectl config get-contexts
```

---

## 11. Storage Classes สำหรับ Cloud Providers

### 11.1 AWS EBS

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:us-east-1:123456789:key/abc123"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### 11.2 GCP Persistent Disk

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### 11.3 Azure Disk

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  kind: Managed
  cachingMode: ReadOnly
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

---

## 12. Monitoring Stack

### 12.1 Prometheus ServiceMonitor

```yaml
# servicemonitor.yaml
---
# Monitor PostgreSQL
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: postgresql-monitor
  namespace: monitoring
  labels:
    app: postgresql
spec:
  namespaceSelector:
    matchNames:
    - databases
  selector:
    matchLabels:
      app: postgresql
  endpoints:
  - port: metrics
    interval: 30s
    scrapeTimeout: 10s
    path: /metrics

---
# Monitor Redis
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: redis-monitor
  namespace: monitoring
spec:
  namespaceSelector:
    matchNames:
    - databases
  selector:
    matchLabels:
      app: redis
  endpoints:
  - port: metrics
    interval: 30s
```

### 12.2 PrometheusRule สำหรับ Alerts

```yaml
# alert-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: database-alerts
  namespace: monitoring
spec:
  groups:
  - name: postgresql.rules
    rules:
    - alert: PostgreSQLDown
      expr: pg_up == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "PostgreSQL instance is down"
        description: "PostgreSQL {{ $labels.instance }} is not responding"
        
    - alert: PostgreSQLTooManyConnections
      expr: sum(pg_stat_activity_count) by (instance) > 250
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "PostgreSQL has too many connections"
        
    - alert: PostgreSQLReplicationLag
      expr: pg_replication_lag > 30
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "PostgreSQL replication lag is too high"
        description: "Replication lag is {{ $value }} seconds"
        
  - name: redis.rules
    rules:
    - alert: RedisDown
      expr: redis_up == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Redis instance is down"
        
    - alert: RedisMemoryUsageHigh
      expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.9
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Redis memory usage is high (>90%)"
```

---

## 13. สรุป

การ deploy Database Cluster บน Kubernetes ต้องคำนึงถึง:

1. **StatefulSet** สำหรับ stateful workloads (databases) เพื่อให้ได้ stable identity และ ordered deployment
2. **PersistentVolumeClaim** เพื่อ retain data แม้ Pod จะ restart
3. **ConfigMap** สำหรับ configuration files
4. **Secret** สำหรับ sensitive data เช่น passwords
5. **Headless Service** สำหรับ DNS-based service discovery
6. **AntiAffinity** เพื่อกระจาย Pods ไปยัง Nodes ต่างๆ
7. **Resource Limits** เพื่อป้องกัน resource starvation
8. **PodDisruptionBudget** เพื่อความพร้อมใช้งานระหว่าง maintenance
9. **NetworkPolicy** เพื่อความปลอดภัยในการเข้าถึง
10. **Monitoring** ด้วย Prometheus/Grafana

### Checklist ก่อน Production

```bash
# ตรวจสอบ PVC bound
kubectl get pvc -n databases | grep -v Bound

# ตรวจสอบ Pods running
kubectl get pods -n databases | grep -v Running

# ตรวจสอบ Services
kubectl get svc -n databases

# Test database connection
kubectl run test-pod --image=postgres:15 --rm -it -- \
  psql -h postgresql-primary.databases.svc.cluster.local \
  -U appuser -d appdb -c "SELECT version();"

# ตรวจสอบ resource usage
kubectl top pods -n databases

# ตรวจสอบ PDB
kubectl get pdb -n databases
```
