# Kubernetes for Production

> **Category:** Infrastructure
> **Version:** 1.0.0
> **Level:** Staff Engineer

---

## Table of Contents

1. [Core Concepts Refresher](#1-core-concepts-refresher)
2. [Production Deployment Patterns](#2-production-deployment-patterns)
3. [Resource Management](#3-resource-management)
4. [Health Probes](#4-health-probes)
5. [Autoscaling](#5-autoscaling)
6. [Networking and Ingress](#6-networking-and-ingress)
7. [Secrets Management](#7-secrets-management)
8. [Pod Security](#8-pod-security)
9. [Storage Patterns](#9-storage-patterns)
10. [Observability in Kubernetes](#10-observability-in-kubernetes)
11. [Zero-Downtime Deployments](#11-zero-downtime-deployments)
12. [Troubleshooting Guide](#12-troubleshooting-guide)

---

## 1. Core Concepts Refresher

```
Cluster    → Group of nodes (VMs) running Kubernetes
Node       → A single VM or physical machine in the cluster
Pod        → Smallest deployable unit; one or more containers sharing network/storage
Deployment → Manages ReplicaSet; ensures desired number of pod replicas
Service    → Stable network endpoint for pods (load balances across pod replicas)
Ingress    → HTTP/HTTPS routing rules; sends traffic to Services

Controllers:
  Deployment   → Stateless apps (API, frontend); rolling updates
  StatefulSet  → Stateful apps (databases, Kafka); stable network identity + storage
  DaemonSet    → One pod per node (log collectors, monitoring agents)
  Job          → Run-to-completion tasks
  CronJob      → Scheduled jobs

Namespaces: logical isolation within a cluster
  production, staging, monitoring, kube-system
  Resource quotas + network policies scoped per namespace
```

---

## 2. Production Deployment Patterns

### Complete API Deployment

```yaml
# api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: production
  labels:
    app: api
    version: "1.2.3"
spec:
  replicas: 3
  
  selector:
    matchLabels:
      app: api
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Allow 1 extra pod during rollout (4 pods max during update)
      maxUnavailable: 0  # Never remove a pod until replacement is ready
  
  template:
    metadata:
      labels:
        app: api
        version: "1.2.3"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/metrics"
        prometheus.io/port: "3000"
    
    spec:
      # Spread pods across nodes and zones for HA
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: api
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: api
      
      # Prefer not to schedule on same node as DB pods
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
                labelSelector:
                  matchLabels:
                    app: postgres
      
      # Graceful termination
      terminationGracePeriodSeconds: 60
      
      # Don't mount default service account token (security)
      automountServiceAccountToken: false
      
      containers:
        - name: api
          image: ghcr.io/gu1lh4s/api:1.2.3  # Always use specific tag, never :latest
          imagePullPolicy: IfNotPresent
          
          ports:
            - containerPort: 3000
              name: http
              protocol: TCP
          
          # Environment variables (non-sensitive)
          env:
            - name: NODE_ENV
              value: production
            - name: PORT
              value: "3000"
            - name: SERVICE_NAME
              value: api
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          
          # Secrets: from Secret object (not hardcoded)
          envFrom:
            - secretRef:
                name: api-secrets  # DATABASE_URL, JWT_SECRET, SMTP_PASSWORD
          
          # Resource limits (REQUIRED for production)
          resources:
            requests:
              cpu: "250m"      # 0.25 CPU guaranteed
              memory: "256Mi"  # 256MB guaranteed
            limits:
              cpu: "1"         # 1 CPU max (throttled if exceeded)
              memory: "512Mi"  # 512MB max (OOMKilled if exceeded)
          
          # Health probes (see section 4)
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          
          # Security context
          securityContext:
            runAsNonRoot: true
            runAsUser: 1001
            runAsGroup: 1001
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities:
              drop: [ALL]
          
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      
      volumes:
        - name: tmp
          emptyDir: {}  # In-memory writable temp dir (required when root is read-only)
      
      # Image pull secret (for private registries)
      imagePullSecrets:
        - name: ghcr-credentials
```

### Service

```yaml
# api-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: production
spec:
  selector:
    app: api
  ports:
    - name: http
      port: 80
      targetPort: 3000
      protocol: TCP
  type: ClusterIP  # Internal only; exposed via Ingress
```

---

## 3. Resource Management

```yaml
# Namespace ResourceQuota — prevent one team from starving others
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"         # Total CPU requests across all pods
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    count/pods: "50"           # Max 50 pods in namespace
    count/services: "20"
    count/persistentvolumeclaims: "10"

---
# LimitRange — default limits for pods that don't specify
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:           # Applied if container doesn't specify limits
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:    # Applied if container doesn't specify requests
        cpu: "100m"
        memory: "128Mi"
      max:               # Maximum allowed (prevents misconfigured huge requests)
        cpu: "4"
        memory: "4Gi"
```

### Right-Sizing Guidelines

```
CPU sizing:
  Request = steady-state usage (what you need guaranteed)
  Limit = burst ceiling (protect the node from run-away processes)
  
  Rule of thumb: request = 50-75% of average; limit = 2-4x request
  
  Too low request: pod gets throttled; poor performance
  Too high request: wastes cluster capacity; fewer pods fit per node
  Too low limit: CPU throttling even on burst (can cause latency spikes)

Memory sizing:
  Request = steady-state; Limit = max allowed
  
  CRITICAL: memory is not compressible
  → If pod exceeds limit: OOMKilled (crash), not throttled
  → Memory leaks become visible very quickly
  
  Set limit = 1.5-2x request
  Monitor actual usage and adjust

CPU Throttling detection:
  kubectl top pods -n production
  kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/production/pods | jq .
  
  In Prometheus: container_cpu_throttled_seconds_total
```

---

## 4. Health Probes

```
Three types of probes:
  livenessProbe:  Is the container alive? If failing → restart container
  readinessProbe: Is the container ready to receive traffic? If failing → remove from Service endpoints
  startupProbe:   Is the container done starting? Delays liveness check until startup completes
  
Common mistake: conflating liveness and readiness
  - Readiness failing = temporarily remove from load balancer (don't restart!)
  - Liveness failing = something is broken, must restart
  
  A database being temporarily down → readiness fails, liveness passes
  The pod should NOT restart just because DB is down
  The pod SHOULD be removed from LB so requests don't hit it
```

```yaml
# Correct probe configuration
livenessProbe:   # Only fail if the process is truly stuck/corrupted
  httpGet:
    path: /health          # Just: { status: 'alive' }
    port: 3000
  initialDelaySeconds: 10  # Wait before first check
  periodSeconds: 30
  timeoutSeconds: 5
  failureThreshold: 3      # 3 consecutive failures → restart (3 × 30s = 90s tolerance)

readinessProbe:  # Fail if dependencies (DB, cache) are unavailable
  httpGet:
    path: /health/ready    # Checks: DB connection, Redis, etc.
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

startupProbe:    # Use for slow-starting containers (DB migrations, JVM warmup)
  httpGet:
    path: /health
    port: 3000
  failureThreshold: 30     # Allow 30 × 10s = 5 minutes for startup
  periodSeconds: 10
  # liveness/readiness probes paused until startupProbe succeeds
```

---

## 5. Autoscaling

### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  
  minReplicas: 3
  maxReplicas: 20
  
  metrics:
    # CPU utilization (most common)
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70  # Scale up when avg CPU > 70% of request
    
    # Memory (less common — memory leaks can cause uncontrolled scale-up)
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    
    # Custom metric: requests per second (requires Prometheus adapter)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"  # Scale up when avg > 100 req/s per pod
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # Wait 60s before scaling up again
      policies:
        - type: Pods
          value: 2                     # Add max 2 pods at a time
          periodSeconds: 60
    
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down (avoid flapping)
      policies:
        - type: Pods
          value: 1                     # Remove max 1 pod at a time
          periodSeconds: 60
```

### Vertical Pod Autoscaler (VPA)

```yaml
# VPA: automatically adjusts CPU/memory requests based on actual usage
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  
  updatePolicy:
    updateMode: "Off"    # Off = only recommend, don't apply (safe for production)
    # "Auto" = apply recommendations + restart pods (can cause brief downtime)
    # "Initial" = apply only on new pods
  
  resourcePolicy:
    containerPolicies:
      - containerName: api
        maxAllowed:
          cpu: "2"
          memory: "1Gi"
        minAllowed:
          cpu: "100m"
          memory: "128Mi"
```

---

## 6. Networking and Ingress

### Ingress (Nginx)

```yaml
# api-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    
    # TLS: cert-manager (automatic Let's Encrypt)
    cert-manager.io/cluster-issuer: letsencrypt-prod
    
    # Rate limiting (nginx annotations)
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "50"
    
    # Security
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    
    # Timeouts
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "5"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    
    # Request size (for file uploads)
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.example.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, PATCH, DELETE, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "Authorization, Content-Type"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"

spec:
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-cert  # cert-manager creates this Secret
  
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```

### Network Policy

```yaml
# Only allow specific traffic flows (default deny all)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  
  policyTypes: [Ingress, Egress]
  
  ingress:
    # Allow traffic from nginx ingress controller
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - port: 3000
    
    # Allow Prometheus scraping from monitoring namespace
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - port: 3000
  
  egress:
    # Allow DB access
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - port: 5432
    
    # Allow Redis
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - port: 6379
    
    # Allow DNS
    - to: []
      ports:
        - port: 53
          protocol: UDP
```

---

## 7. Secrets Management

```yaml
# DO NOT store secrets in plain YAML in git
# Options:
# 1. Sealed Secrets (kubeseal): encrypt secrets, commit encrypted form
# 2. External Secrets Operator: pull from AWS Secrets Manager / GCP Secret Manager / Vault
# 3. Vault Agent Injector: inject secrets at runtime via sidecar

# Option 1: External Secrets Operator (with AWS Secrets Manager)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: api-secrets
  namespace: production
spec:
  refreshInterval: 1h
  
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  
  target:
    name: api-secrets  # Creates this Kubernetes Secret
    creationPolicy: Owner
  
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: production/api
        property: database_url
    
    - secretKey: JWT_SECRET
      remoteRef:
        key: production/api
        property: jwt_secret
    
    - secretKey: SMTP_PASSWORD
      remoteRef:
        key: production/api
        property: smtp_password
```

---

## 8. Pod Security

```yaml
# Pod Security Standards (replace deprecated PodSecurityPolicy)
# Apply to namespace via label

# Labels on namespace:
# pod-security.kubernetes.io/enforce: restricted
# pod-security.kubernetes.io/warn: restricted
# pod-security.kubernetes.io/audit: restricted

# Kyverno policy: enforce secure defaults
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-pod-security
spec:
  validationFailureAction: Enforce
  rules:
    - name: no-privileged-containers
      match:
        resources:
          kinds: [Pod]
      validate:
        message: "Privileged containers are not allowed"
        pattern:
          spec:
            containers:
              - (securityContext):
                  =(privileged): "false"
    
    - name: require-non-root
      match:
        resources:
          kinds: [Pod]
      validate:
        message: "Containers must run as non-root"
        pattern:
          spec:
            securityContext:
              runAsNonRoot: true
    
    - name: require-requests-limits
      match:
        resources:
          kinds: [Pod]
      validate:
        message: "CPU and memory requests/limits required"
        pattern:
          spec:
            containers:
              - resources:
                  requests:
                    memory: "?*"
                    cpu: "?*"
                  limits:
                    memory: "?*"
                    cpu: "?*"
```

---

## 9. Storage Patterns

```yaml
# StatefulSet for databases
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres
  replicas: 1  # Or 3 for HA with Patroni/Citus
  
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
          image: postgres:16.1-alpine
          
          env:
            - name: POSTGRES_DB
              value: appdb
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: username
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
          
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  
  # Persistent storage (not deleted when pod restarts)
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: gp3          # AWS: gp3 SSD
        resources:
          requests:
            storage: 100Gi
```

---

## 10. Observability in Kubernetes

```bash
# Essential kubectl debugging commands

# Pod status and events
kubectl get pods -n production -o wide
kubectl describe pod api-xxx-yyy -n production  # Shows events, probe failures
kubectl logs api-xxx-yyy -n production          # Current logs
kubectl logs api-xxx-yyy -n production --previous  # Logs from crashed container

# Resource usage
kubectl top pods -n production
kubectl top nodes

# Exec into running container
kubectl exec -it api-xxx-yyy -n production -- sh

# Port forward for local debugging
kubectl port-forward pod/api-xxx-yyy 3000:3000 -n production

# Watch deployment rollout
kubectl rollout status deployment/api -n production
kubectl rollout history deployment/api -n production

# Emergency: rollback to previous version
kubectl rollout undo deployment/api -n production
```

---

## 11. Zero-Downtime Deployments

```bash
# Zero-downtime deployment checklist:

# 1. readinessProbe must be configured
#    New pod must pass readiness before receiving traffic

# 2. terminationGracePeriodSeconds must cover request timeout
#    Set to: max request processing time + 10s buffer

# 3. preStop hook: drain connections before SIGTERM
```

```yaml
# PreStop hook gives Kubernetes time to remove pod from load balancer
# before sending SIGTERM (avoids split-brain with Ingress)
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]  # Wait 10s for LB to drain

# Full graceful shutdown flow:
# 1. Pod marked NotReady → removed from Service endpoints
# 2. preStop hook runs (sleep 10 → LB finishes draining)
# 3. SIGTERM sent to container
# 4. App handles SIGTERM → stop accepting new requests, finish in-flight
# 5. terminationGracePeriodSeconds countdown starts
# 6. SIGKILL after grace period if still running
```

```bash
# Blue-Green via namespace
kubectl apply -f deployment-v2.yaml
kubectl rollout status deployment/api-v2 -n production

# Canary via traffic splitting (with nginx annotations or Argo Rollouts)
kubectl annotate ingress api-ingress \
  nginx.ingress.kubernetes.io/canary: "true" \
  nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% to canary
```

---

## 12. Troubleshooting Guide

```
Pod stuck in "Pending":
  1. kubectl describe pod xxx → look at Events section
  2. Common causes:
     - Insufficient CPU/memory: "0/3 nodes are available: 3 Insufficient cpu"
       → Reduce resource requests or scale up nodes
     - PVC not bound: storage class issue or quota exceeded
     - Taint/toleration mismatch: pod can't be scheduled on available nodes
     - ImagePullBackOff: registry credentials or wrong image tag

Pod stuck in "CrashLoopBackOff":
  1. kubectl logs xxx --previous → crash reason
  2. kubectl describe pod xxx → exit code
     - Exit 137: OOMKilled (increase memory limit)
     - Exit 1: App crashed (check logs)
     - Exit 143: SIGTERM not handled properly (fix graceful shutdown)

Pod failing readinessProbe:
  1. kubectl exec -it xxx -- curl http://localhost:3000/health/ready
  2. Check if DB/Redis is reachable from inside pod
  3. Check environment variables are set: kubectl exec -it xxx -- env | grep DB

Service not routing to pods:
  1. kubectl get endpoints api -n production → should show pod IPs
     If empty: selector labels don't match pod labels
  2. kubectl get pods -n production --show-labels → verify labels
  3. Test directly: kubectl exec debug-pod -- curl http://api/health

Slow API response:
  1. Check pod CPU throttling: kubectl top pods
  2. Check DB slow queries: kubectl logs postgres-xxx | grep duration
  3. Check HPA: kubectl get hpa api -n production → is it at maxReplicas?
  4. Check node pressure: kubectl describe nodes | grep -A5 Conditions
```
