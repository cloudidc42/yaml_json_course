# Part 81: Performance Tuning and Optimization
## Steps 771-780: Resource Sizing, JVM Tuning, Node Efficiency

---

## Step 771: Performance Optimization Overview

```
Kubernetes Performance Tuning Areas:

1. Cluster-level:
   - API server: request rate limits, watch cache
   - etcd: SSD storage, defragmentation
   - kube-proxy: IPVS mode (vs iptables)
   - CoreDNS: caching, ndots optimization

2. Node-level:
   - CPU pinning (NUMA-aware scheduling)
   - Huge pages for memory-intensive apps
   - Container runtime tuning (containerd)
   - Node local DNS cache

3. Workload-level:
   - Right-sized resource requests/limits
   - JVM tuning for Java apps
   - Connection pooling
   - Pod topology (anti-affinity, topology spread)

4. Network-level:
   - Service type selection
   - IPVS for large-scale services
   - Session affinity for stateful apps
   - Topology-aware routing (reduce cross-zone traffic)

5. Storage-level:
   - StorageClass: gp3 vs gp2 (AWS)
   - Local volumes for database I/O
   - PVC pre-warming
```

---

## Step 772: Right-Sizing with VPA Recommendations

```yaml
# Deploy VPA in Off mode to collect recommendations
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-rightsizing
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Off"  # Observe only

# After 7-14 days:
# kubectl describe vpa myapp-rightsizing -n production
# Use Target recommendation for requests
# Use Upper Bound for limits

---
# Apply VPA recommendations to deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  template:
    spec:
      containers:
        - name: app
          resources:
            requests:
              cpu: 100m     # VPA Target
              memory: 256Mi
            limits:
              cpu: 400m     # 4x request (burst headroom)
              memory: 512Mi # VPA Upper Bound
```

---

## Step 773: JVM Tuning for Java Applications

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
  namespace: production
spec:
  template:
    spec:
      containers:
        - name: app
          image: my-java-app:1.0
          env:
            - name: JAVA_OPTS
              value: >-
                -XX:+UseContainerSupport
                -XX:MaxRAMPercentage=75.0
                -XX:InitialRAMPercentage=50.0
                -XX:+UseG1GC
                -XX:MaxGCPauseMillis=200
                -XX:+ExitOnOutOfMemoryError
                -XX:+HeapDumpOnOutOfMemoryError
                -XX:HeapDumpPath=/tmp/heapdump.hprof
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 2000m
              memory: 2Gi  # JVM heap = 75% = 1.5Gi
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60  # JVM startup time
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            failureThreshold: 30  # 5 min max startup time
            periodSeconds: 10
```

---

## Step 774: CoreDNS Performance Tuning

```yaml
# Pod: reduce DNS ndots for faster resolution
apiVersion: v1
kind: Pod
metadata:
  name: optimized-pod
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "2"        # Default is 5 (causes 5 extra lookups)
      - name: single-request-reopen
      - name: timeout
        value: "1"
  containers:
    - name: app
      image: myapp:1.0

---
# CoreDNS caching config
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
      errors
      health
      ready
      kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
      }
      cache {
        success 9984 30  # Cache 9984 entries, 30s TTL
        denial 9984 5
        prefetch 10      # Prefetch popular entries
      }
      prometheus :9153
      forward . /etc/resolv.conf {
        max_concurrent 1000
      }
      loop
      reload
      loadbalance
    }
```

---

## Step 775: CPU and Memory QoS

```yaml
# Guaranteed QoS: requests == limits
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
    - name: app
      resources:
        requests:
          cpu: 1000m
          memory: 1Gi
        limits:
          cpu: 1000m    # Same as request = Guaranteed
          memory: 1Gi

---
# CPU Manager: dedicated CPUs for critical pods
# kubelet: --cpu-manager-policy=static
apiVersion: v1
kind: Pod
metadata:
  name: cpu-pinned-pod
spec:
  containers:
    - name: app
      resources:
        requests:
          cpu: "2"    # Integer = static CPU pinning
          memory: 2Gi
        limits:
          cpu: "2"    # Must equal request for Guaranteed
          memory: 2Gi

---
# Huge pages for databases
apiVersion: v1
kind: Pod
metadata:
  name: hugepages-db
spec:
  containers:
    - name: database
      image: postgres:15
      resources:
        requests:
          hugepages-2Mi: 1Gi
          memory: 2Gi
        limits:
          hugepages-2Mi: 1Gi
          memory: 2Gi
      volumeMounts:
        - name: hugepage
          mountPath: /dev/hugepages
  volumes:
    - name: hugepage
      emptyDir:
        medium: HugePages-2Mi
```

---

## Step 776: IPVS for kube-proxy Performance

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-proxy
  namespace: kube-system
data:
  config.conf: |
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    kind: KubeProxyConfiguration
    mode: ipvs
    ipvs:
      scheduler: rr      # Round-robin
      syncPeriod: 30s
      minSyncPeriod: 5s
      strictARP: true
    conntrack:
      maxPerCore: 131072
      min: 131072
      tcpEstablishedTimeout: 86400s

# Verify: ipvsadm -Ln | head -20
```

---

## Step 777: Topology Spread Constraints

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: well-spread-app
  namespace: production
spec:
  replicas: 9
  template:
    metadata:
      labels:
        app: well-spread-app
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: well-spread-app
        
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: well-spread-app
      
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values: [well-spread-app]
              topologyKey: kubernetes.io/hostname
      
      containers:
        - name: app
          image: myapp:1.0
```

---

## Step 778: Zero-Downtime Deployments

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graceful-app
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0    # Never take pods down first
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: app
          image: myapp:1.0
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]  # Wait for LB deregistration
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            successThreshold: 1
            failureThreshold: 3
            periodSeconds: 5
```

---

## Step 779: Connection Pooling

```yaml
# PgBouncer: connection pooler for PostgreSQL
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgbouncer
  namespace: production
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: pgbouncer
          image: pgbouncer/pgbouncer:1.21
          env:
            - name: POSTGRESQL_HOST
              value: postgres.databases.svc.cluster.local
            - name: POOL_MODE
              value: transaction
            - name: MAX_CLIENT_CONN
              value: "1000"
            - name: DEFAULT_POOL_SIZE
              value: "20"
          ports:
            - containerPort: 5432
```

---

## Step 780: Workshop - Performance Alerts

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: performance-alerts
  namespace: monitoring
spec:
  groups:
    - name: performance
      rules:
        - alert: HighCPUThrottling
          expr: |
            rate(container_cpu_cfs_throttled_seconds_total[5m])
            /
            rate(container_cpu_cfs_periods_total[5m])
            > 0.5
          for: 10m
          annotations:
            summary: "Container {{ $labels.container }} CPU throttling > 50%"
          labels:
            severity: warning

        - alert: HighMemoryUsage
          expr: |
            container_memory_working_set_bytes
            /
            kube_pod_container_resource_limits{resource="memory"}
            > 0.9
          for: 5m
          annotations:
            summary: "Container {{ $labels.container }} memory > 90% of limit"
          labels:
            severity: warning

        - alert: PodsNotSpreadAcrossZones
          expr: |
            max(kube_pod_info) by (node, zone, namespace, pod)
            - min(kube_pod_info) by (zone, namespace)
            > 2
          for: 15m
          annotations:
            summary: "Pods in {{ $labels.namespace }} unevenly spread across zones"
          labels:
            severity: info
```

---

## 📊 สรุป Part 81

| Area | Tool/Technique | Impact |
|------|---------------|--------|
| Resource sizing | VPA Recommendations | Reduce waste 30-50% |
| JVM tuning | UseContainerSupport + G1GC | Reduce OOM/GC pauses |
| DNS | NodeLocal DNSCache, ndots=2 | -50% DNS latency |
| kube-proxy | IPVS mode | 10x more services |
| Spread | TopologySpreadConstraints | HA + reduce cross-AZ cost |
| Startup | PreStop + maxUnavailable=0 | Zero-downtime deploys |

---

## 🔗 ต่อไป
- [Part 82: Cloud-Native Patterns](./part-82-cloud-native-patterns.md)

---
*Part 81 | Steps 771-780 | Performance Tuning | Educational Content*
