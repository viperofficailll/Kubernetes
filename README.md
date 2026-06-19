# Kubernetes for Node.js Microservices — Complete Guide to Deploying on AWS (EKS)

> Built around a real microservices repo structure: `users-service`, `order-service`, `cart-service`, `notification-service`, `restaurant-service`, `serveio-frontend`. Every Kubernetes concept explained, then deployed end-to-end on AWS EKS.

---

## Table of Contents

1. [Why Kubernetes for Microservices?](#1-why-kubernetes-for-microservices)
2. [Core Kubernetes Components](#2-core-kubernetes-components)
   - 2.1 [Cluster, Node, and Control Plane](#21-cluster-node-and-control-plane)
   - 2.2 [Pod](#22-pod)
   - 2.3 [Deployment](#23-deployment)
   - 2.4 [ReplicaSet](#24-replicaset)
   - 2.5 [Service](#25-service)
   - 2.6 [Ingress](#26-ingress)
   - 2.7 [ConfigMap](#27-configmap)
   - 2.8 [Secret](#28-secret)
   - 2.9 [Namespace](#29-namespace)
   - 2.10 [PersistentVolume & PersistentVolumeClaim](#210-persistentvolume--persistentvolumeclaim)
   - 2.11 [HorizontalPodAutoscaler (HPA)](#211-horizontalpodautoscaler-hpa)
   - 2.12 [Liveness, Readiness & Startup Probes](#212-liveness-readiness--startup-probes)
   - 2.13 [Jobs & CronJobs](#213-jobs--cronjobs)
   - 2.14 [StatefulSet](#214-statefulset)
3. [How a Request Flows Through the Cluster](#3-how-a-request-flows-through-the-cluster)
4. [Mapping the Repo Structure to Kubernetes](#4-mapping-the-repo-structure-to-kubernetes)
5. [Project Setup — Dockerizing Each Node.js Service](#5-project-setup--dockerizing-each-nodejs-service)
6. [Kubernetes Manifests Per Service](#6-kubernetes-manifests-per-service)
   - 6.1 [Namespace](#61-namespace)
   - 6.2 [ConfigMap & Secrets](#62-configmap--secrets)
   - 6.3 [users-service](#63-users-service)
   - 6.4 [order-service](#64-order-service)
   - 6.5 [cart-service](#65-cart-service)
   - 6.6 [notification-service](#66-notification-service)
   - 6.7 [restaurant-service](#67-restaurant-service)
   - 6.8 [serveio-frontend](#68-serveio-frontend)
   - 6.9 [Ingress — routing all services](#69-ingress--routing-all-services)
7. [Setting Up AWS Infrastructure](#7-setting-up-aws-infrastructure)
   - 7.1 [Prerequisites](#71-prerequisites)
   - 7.2 [Creating the EKS Cluster](#72-creating-the-eks-cluster)
   - 7.3 [Setting Up ECR (Container Registry)](#73-setting-up-ecr-container-registry)
   - 7.4 [Installing the AWS Load Balancer Controller](#74-installing-the-aws-load-balancer-controller)
   - 7.5 [RDS, ElastiCache, and Amazon MQ](#75-rds-elasticache-and-amazon-mq)
8. [CI/CD — GitHub Actions to EKS](#8-cicd--github-actions-to-eks)
9. [Helm — Packaging Your Services](#9-helm--packaging-your-services)
10. [Observability — Logs, Metrics, Tracing](#10-observability--logs-metrics-tracing)
11. [Scaling Strategies](#11-scaling-strategies)
12. [Security Checklist](#12-security-checklist)
13. [Common Pitfalls](#13-common-pitfalls)

---

## 1. Why Kubernetes for Microservices?

Looking at the repo structure — `users-service`, `order-service`, `cart-service`, `notification-service`, `restaurant-service` — this is a classic microservices architecture. Each service:

- Has its own codebase, deploy cycle, and scaling needs
- Needs to discover and call other services (`order-service` → `notification-service`)
- Needs config and secrets (DB URLs, JWT secrets, RabbitMQ URLs)
- Needs to survive crashes, scale under load, and roll out updates with zero downtime

**Running these manually with `docker run` doesn't solve any of that.** Kubernetes gives you:

| Problem | Kubernetes Solution |
|---|---|
| Service crashes | Self-healing — restarts automatically |
| Traffic spikes | Horizontal Pod Autoscaler |
| Service discovery | Built-in DNS (`order-service.default.svc.cluster.local`) |
| Zero-downtime deploys | Rolling updates |
| Config differs per environment | ConfigMaps + Secrets |
| Load balancing across replicas | Service object + kube-proxy |
| Routing `/api/orders` vs `/api/cart` | Ingress |

```
┌─────────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster (EKS)                    │
│                                                                  │
│  Ingress (ALB) ──┬──▶ users-service        (3 pods)             │
│                  ├──▶ order-service         (3 pods)             │
│                  ├──▶ cart-service          (2 pods)             │
│                  ├──▶ notification-service  (2 pods)             │
│                  ├──▶ restaurant-service    (2 pods)             │
│                  └──▶ serveio-frontend      (2 pods)             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Core Kubernetes Components

### 2.1 Cluster, Node, and Control Plane

A **cluster** is the whole Kubernetes system — a set of machines running your containers plus the brains that manage them.

```
┌──────────────────────── CLUSTER ─────────────────────────────┐
│                                                              │
│  ┌─── CONTROL PLANE (managed by AWS in EKS) ─────────────┐  │
│  │  API Server  │  Scheduler  │  Controller Manager  │etcd │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──── NODE 1 ────┐  ┌──── NODE 2 ────┐  ┌──── NODE 3 ────┐ │
│  │ kubelet        │  │ kubelet        │  │ kubelet        │ │
│  │ kube-proxy     │  │ kube-proxy     │  │ kube-proxy     │ │
│  │ [pod] [pod]    │  │ [pod] [pod]    │  │ [pod] [pod]    │ │
│  └────────────────┘  └────────────────┘  └────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

- **Node** — a worker machine (EC2 instance in EKS) that runs your pods
- **Control Plane** — the API server, scheduler, and etcd database that decide *what* runs *where*. In EKS, AWS manages this for you — you never SSH into it.
- **kubelet** — the agent on each node that talks to the control plane and starts/stops containers
- **kube-proxy** — handles network routing so Services can load-balance to pods

### 2.2 Pod

A **Pod** is the smallest deployable unit — one or more containers that share networking and storage, always scheduled together on the same node.

```yaml
# A bare pod (you rarely create these directly — Deployments manage them)
apiVersion: v1
kind: Pod
metadata:
  name: order-service-pod
spec:
  containers:
    - name: order-service
      image: myregistry/order-service:1.0.0
      ports:
        - containerPort: 3000
```

**Key facts:**
- Each pod gets its own IP address (ephemeral — changes if the pod restarts)
- Containers in the same pod share `localhost` networking
- Pods are disposable — Kubernetes can kill and recreate them anytime
- You almost never create bare Pods — use a **Deployment** instead

### 2.3 Deployment

A **Deployment** manages a set of identical pod replicas. It handles rolling updates, rollbacks, and self-healing.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3                          # run 3 identical pods
  selector:
    matchLabels:
      app: order-service
  template:                            # the pod "stamp" — every replica looks like this
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: myregistry/order-service:1.0.0
          ports:
            - containerPort: 3000
  strategy:
    type: RollingUpdate                # zero-downtime deploys
    rollingUpdate:
      maxSurge: 1                      # can create 1 extra pod during rollout
      maxUnavailable: 0                # never go below the desired count
```

**What happens when you update the image:**

```
Before:  [v1] [v1] [v1]
Step 1:  [v1] [v1] [v1] [v2]   ← maxSurge: 1 extra pod created
Step 2:  [v1] [v1] [v2]        ← one v1 terminated once v2 is healthy
Step 3:  [v1] [v2] [v2]
Step 4:  [v2] [v2] [v2]        ← rollout complete
```

```bash
# Trigger a rolling update
kubectl set image deployment/order-service order-service=myregistry/order-service:1.1.0

# Watch the rollout
kubectl rollout status deployment/order-service

# Something broke? Roll back instantly
kubectl rollout undo deployment/order-service
```

### 2.4 ReplicaSet

A **ReplicaSet** ensures a specified number of pod replicas are running at all times. You rarely write these directly — Deployments create and manage ReplicaSets for you automatically (one ReplicaSet per version, enabling rollback).

```
Deployment "order-service"
  ├── ReplicaSet (v1.0.0) — scaled to 0 after rollout
  └── ReplicaSet (v1.1.0) — scaled to 3 (current)
```

### 2.5 Service

A **Service** is a stable network endpoint that load-balances traffic across a set of pods, even as individual pods come and go. Pods are ephemeral (their IPs change) — Services give you a permanent address.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service             # DNS name: order-service.default.svc.cluster.local
spec:
  type: ClusterIP                  # internal-only (default)
  selector:
    app: order-service             # routes to any pod with this label
  ports:
    - port: 3000                   # port the Service listens on
      targetPort: 3000              # port the container listens on
```

**Service Types:**

| Type | Use Case | Accessible From |
|---|---|---|
| `ClusterIP` (default) | Internal service-to-service calls | Only inside the cluster |
| `NodePort` | Quick testing | `<NodeIP>:<port>` (rarely used in production) |
| `LoadBalancer` | Expose directly to internet | Creates an AWS ELB automatically |
| `ExternalName` | Alias to an external DNS name | Maps to an outside service (e.g., managed RDS) |

```typescript
// Inside order-service's Node.js code, calling notification-service:
// Kubernetes DNS resolves the Service name automatically — no hardcoded IPs.
await axios.post('http://notification-service:4000/notify', {
  orderId: order.id,
  userId:  order.userId,
});
// "notification-service" resolves via cluster DNS to the Service's ClusterIP,
// which load-balances across all healthy notification-service pods.
```

### 2.6 Ingress

An **Ingress** routes external HTTP/HTTPS traffic to the correct internal Service based on hostname or path. It's the "front door" — like an API gateway built into Kubernetes.

```
Internet
   │
   ▼
┌─────────────────────────────────────────────────┐
│         Ingress (backed by AWS ALB)              │
│                                                  │
│  /api/users/*        → users-service             │
│  /api/orders/*       → order-service              │
│  /api/cart/*         → cart-service                │
│  /api/notifications/* → notification-service       │
│  /api/restaurants/*  → restaurant-service          │
│  /*  (everything else) → serveio-frontend           │
└─────────────────────────────────────────────────┘
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: serveio-ingress
  annotations:
    kubernetes.io/ingress.class: alb               # use AWS ALB Ingress Controller
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - host: api.serveio.com
      http:
        paths:
          - path: /api/users
            pathType: Prefix
            backend:
              service:
                name: users-service
                port:
                  number: 3000
          - path: /api/orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 3000
```

### 2.7 ConfigMap

A **ConfigMap** stores non-sensitive configuration data as key-value pairs, decoupled from your container image. Same image, different environments — just swap the ConfigMap.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
data:
  NODE_ENV:        "production"
  PORT:             "3000"
  USERS_SERVICE_URL: "http://users-service:3000"
  LOG_LEVEL:        "info"
```

```yaml
# Consuming it in a Deployment
spec:
  containers:
    - name: order-service
      envFrom:
        - configMapRef:
            name: order-service-config
```

### 2.8 Secret

A **Secret** is like a ConfigMap but for sensitive data (DB passwords, JWT secrets, API keys). Stored base64-encoded by default (NOT encrypted unless you enable encryption at rest).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
type: Opaque
stringData:                      # stringData accepts plain text — k8s base64-encodes it for you
  DATABASE_URL:    "postgresql://user:pass@order-db.xxxx.rds.amazonaws.com:5432/orders"
  JWT_SECRET:      "super-secret-key-change-me"
  RABBITMQ_URL:    "amqp://admin:secret@rabbitmq-service:5672"
```

```bash
# Better: create secrets from the CLI rather than committing them to Git
kubectl create secret generic order-service-secrets \
  --from-literal=DATABASE_URL='postgresql://...' \
  --from-literal=JWT_SECRET='...' \
  -n serveio
```

> **Production tip:** Use **AWS Secrets Manager** + the **External Secrets Operator** instead of raw Kubernetes Secrets — covered in section 12.

### 2.9 Namespace

A **Namespace** is a virtual cluster within a cluster — logical isolation for resources, RBAC, and resource quotas.

```bash
kubectl create namespace serveio-prod
kubectl create namespace serveio-staging

# Deploy to a specific namespace
kubectl apply -f order-service.yaml -n serveio-prod
```

```
Cluster
├── namespace: serveio-prod
│   ├── order-service (3 pods)
│   └── cart-service (2 pods)
├── namespace: serveio-staging
│   ├── order-service (1 pod)
│   └── cart-service (1 pod)
└── namespace: kube-system  (Kubernetes internals)
```

### 2.10 PersistentVolume & PersistentVolumeClaim

Pods are ephemeral — their local disk disappears when they're deleted. For data that must survive (uploaded files, logs), use a **PersistentVolume (PV)** backed by real storage, claimed via a **PersistentVolumeClaim (PVC)**.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: notification-uploads-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3              # AWS EBS-backed storage class
  resources:
    requests:
      storage: 10Gi
---
# Mounted into a pod:
spec:
  containers:
    - name: notification-service
      volumeMounts:
        - name: uploads
          mountPath: /app/uploads
  volumes:
    - name: uploads
      persistentVolumeClaim:
        claimName: notification-uploads-pvc
```

> **Note:** For stateless Node.js microservices, you typically DON'T need PVs — use S3 for file storage and RDS for databases instead. PVs matter more for stateful workloads (databases run inside the cluster, message brokers).

### 2.11 HorizontalPodAutoscaler (HPA)

An **HPA** automatically adjusts the number of pod replicas based on observed CPU/memory usage (or custom metrics).

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70        # scale up when avg CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

```
Load increases → CPU usage rises above 70%
       ↓
HPA controller checks metrics every 15s
       ↓
Scales Deployment from 2 → 4 → 6 replicas (gradual)
       ↓
Load decreases → HPA scales back down (after a cooldown window)
```

```bash
kubectl autoscale deployment order-service --cpu-percent=70 --min=2 --max=10
kubectl get hpa
```

### 2.12 Liveness, Readiness & Startup Probes

Probes tell Kubernetes whether a container is healthy and ready for traffic — critical for Node.js apps that need a moment to connect to the DB on startup.

```yaml
spec:
  containers:
    - name: order-service
      image: myregistry/order-service:1.0.0

      # Is the process alive? If this fails repeatedly, k8s KILLS and RESTARTS the pod.
      livenessProbe:
        httpGet:
          path: /health/live
          port: 3000
        initialDelaySeconds: 10
        periodSeconds:       10
        failureThreshold:    3

      # Is the app ready to receive traffic? If this fails, k8s REMOVES it from
      # the Service's load-balancing pool — but does NOT restart the pod.
      readinessProbe:
        httpGet:
          path: /health/ready
          port: 3000
        initialDelaySeconds: 5
        periodSeconds:       5
        failureThreshold:    3

      # For slow-starting apps — gives extra time before liveness checks begin.
      # Prevents Kubernetes from killing a pod that's still booting (e.g. running migrations).
      startupProbe:
        httpGet:
          path: /health/live
          port: 3000
        failureThreshold: 30          # 30 × 5s = up to 150s to start
        periodSeconds: 5
```

```typescript
// Node.js implementation — Express health endpoints
import express from 'express';
const app = express();

let isReady = false;       // becomes true after DB connection succeeds
let dbConnected = false;

// Liveness: "is the Node.js process alive and responsive?"
// Keep this simple — don't check dependencies here, or a flaky DB
// connection will cause Kubernetes to restart a perfectly healthy pod.
app.get('/health/live', (req, res) => {
  res.status(200).json({ status: 'alive' });
});

// Readiness: "can I actually serve requests right now?"
// Check real dependencies — DB, message broker, etc.
app.get('/health/ready', async (req, res) => {
  if (!dbConnected) {
    return res.status(503).json({ status: 'not_ready', reason: 'database not connected' });
  }
  res.status(200).json({ status: 'ready' });
});

async function startServer() {
  await connectToDatabase();   // sets dbConnected = true on success
  dbConnected = true;
  app.listen(3000, () => console.log('order-service listening on 3000'));
}

startServer();
```

### 2.13 Jobs & CronJobs

A **Job** runs a task to completion (e.g., a database migration). A **CronJob** runs Jobs on a schedule.

```yaml
# One-time DB migration job — run before deploying a new version
apiVersion: batch/v1
kind: Job
metadata:
  name: order-service-migrate
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myregistry/order-service:1.1.0
          command: ["npm", "run", "migrate"]
      restartPolicy: Never
  backoffLimit: 3
---
# Scheduled job — e.g., clean up expired carts every night
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cart-cleanup
spec:
  schedule: "0 2 * * *"              # every day at 2 AM (cron syntax)
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cart-cleanup
              image: myregistry/cart-service:1.0.0
              command: ["node", "scripts/cleanup-expired-carts.js"]
          restartPolicy: OnFailure
```

### 2.14 StatefulSet

A **StatefulSet** is like a Deployment but for stateful apps that need stable network identities and persistent storage per pod (databases, message brokers). You'd use this if you run RabbitMQ or Postgres *inside* the cluster rather than using a managed AWS service (RDS/Amazon MQ) — which is the recommended approach for production, covered in section 7.5.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
spec:
  serviceName: rabbitmq
  replicas: 3
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      containers:
        - name: rabbitmq
          image: rabbitmq:3.13-management
          volumeMounts:
            - name: data
              mountPath: /var/lib/rabbitmq
  volumeClaimTemplates:          # each pod gets ITS OWN PVC — rabbitmq-data-0, rabbitmq-data-1...
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 5Gi
```

---

## 3. How a Request Flows Through the Cluster

```
1. User's browser ──HTTPS──▶ Route53 DNS (api.serveio.com)
                                  │
2.                                ▼
                            AWS Application Load Balancer (ALB)
                            (created automatically by Ingress)
                                  │
3.                                ▼
                            Ingress Controller evaluates path rules
                            "/api/orders/*" → order-service
                                  │
4.                                ▼
                            Service "order-service" (ClusterIP)
                            load-balances across healthy pods
                                  │
5.                  ┌─────────────┼─────────────┐
                     ▼             ▼             ▼
6.              Pod (order-service-abc) Pod (order-service-def) Pod (order-service-ghi)
                Readiness probe: ✅       Readiness probe: ✅      Readiness probe: ❌ (excluded)
                     │
7.                   ▼
              order-service calls users-service via cluster DNS:
              http://users-service:3000/verify-token
                     │
8.                   ▼
              order-service writes to RDS Postgres (outside cluster)
              order-service publishes "order.created" to Amazon MQ (RabbitMQ)
                     │
9.                   ▼
              notification-service consumes from queue → sends push notification
```

---

## 4. Mapping the Repo Structure to Kubernetes

Based on the repo screenshot, here's how each folder becomes a Kubernetes Deployment:

| Repo Folder | K8s Deployment | K8s Service | Notes |
|---|---|---|---|
| `users-service` | `users-service` | ClusterIP | Auth, JWT issuing — needs CORS config (per the commit history) |
| `order-service` | `order-service` | ClusterIP | Calls notification-service on status change |
| `cart-service` | `cart-service` | ClusterIP | Integrated with order via UI (per commits) |
| `notification-service` | `notification-service` | ClusterIP | Consumes from RabbitMQ/Amazon MQ |
| `restaurant-service` | `restaurant-service` | ClusterIP | Menu/restaurant data |
| `serveio-frontend` | `serveio-frontend` | ClusterIP | Served via Ingress as the default backend |
| `docker-compose.yml` | → replaced by K8s manifests | — | Used for local dev; K8s is the prod equivalent |
| `.env.example` | → ConfigMap + Secret | — | Each variable becomes a K8s config/secret key |
| `.github/workflows` | → CI/CD pipeline | — | Builds images, pushes to ECR, applies manifests (Section 8) |

---

## 5. Project Setup — Dockerizing Each Node.js Service

Every service needs a production-ready Dockerfile before it can run in Kubernetes.

### `order-service/Dockerfile`

```dockerfile
# ─── Stage 1: Build ────────────────────────────────────────────────────────────
FROM node:20-alpine AS builder
WORKDIR /app

# Copy dependency manifests first — leverages Docker layer caching.
# If package.json hasn't changed, this layer is reused on rebuild.
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build          # compiles TypeScript → dist/

# ─── Stage 2: Production ───────────────────────────────────────────────────────
FROM node:20-alpine AS production
WORKDIR /app

# Run as non-root user — required for many K8s security policies (see section 12)
RUN addgroup -g 1001 nodejs && adduser -S nodeuser -u 1001

COPY package*.json ./
RUN npm ci --omit=dev      # production deps only — smaller image, faster startup

COPY --from=builder /app/dist ./dist

USER nodeuser
EXPOSE 3000

# Use a process manager-friendly CMD — npm wraps signals poorly,
# call node directly so SIGTERM (sent by k8s on pod termination) reaches your app
CMD ["node", "dist/index.js"]
```

```dockerfile
# .dockerignore — keep build context small
node_modules
dist
.env
.git
*.md
coverage
.github
```

### Graceful Shutdown — Required for Zero-Downtime Deploys

```typescript
// order-service/src/index.ts
/**
 * Without this, Kubernetes sends SIGTERM during a rolling update,
 * Node.js dies INSTANTLY, and any in-flight HTTP requests get dropped.
 *
 * Kubernetes gives a pod `terminationGracePeriodSeconds` (default 30s)
 * to shut down cleanly before force-killing with SIGKILL.
 */
import express from 'express';
import { createServer } from 'http';

const app = express();
// ... routes ...

const server = createServer(app);
server.listen(3000, () => console.log('order-service listening on 3000'));

let isShuttingDown = false;

async function gracefulShutdown(signal: string) {
  console.log(`[Shutdown] Received ${signal}`);
  isShuttingDown = true;

  // Tell the readiness probe to start failing IMMEDIATELY.
  // This removes the pod from the Service's load-balancing pool
  // before we actually stop accepting connections — avoids dropped requests.

  server.close(async () => {
    console.log('[Shutdown] HTTP server closed — no new connections accepted');

    await closeDbConnections();
    await closeRabbitMQConnection();

    console.log('[Shutdown] Cleanup complete — exiting');
    process.exit(0);
  });

  // Failsafe: force exit if cleanup hangs longer than the grace period
  setTimeout(() => {
    console.error('[Shutdown] Forced exit after timeout');
    process.exit(1);
  }, 25_000); // stay under terminationGracePeriodSeconds (30s default)
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT',  () => gracefulShutdown('SIGINT'));

// Wire isShuttingDown into the readiness probe
app.get('/health/ready', (req, res) => {
  if (isShuttingDown) {
    return res.status(503).json({ status: 'shutting_down' });
  }
  res.status(200).json({ status: 'ready' });
});
```

---

## 6. Kubernetes Manifests Per Service

### 6.1 Namespace

```yaml
# k8s/00-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: serveio
  labels:
    env: production
```

```bash
kubectl apply -f k8s/00-namespace.yaml
```

### 6.2 ConfigMap & Secrets

```yaml
# k8s/01-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: serveio-shared-config
  namespace: serveio
data:
  NODE_ENV: "production"
  USERS_SERVICE_URL:        "http://users-service:3000"
  ORDER_SERVICE_URL:        "http://order-service:3000"
  CART_SERVICE_URL:         "http://cart-service:3000"
  NOTIFICATION_SERVICE_URL: "http://notification-service:3000"
  RESTAURANT_SERVICE_URL:   "http://restaurant-service:3000"
```

```bash
# Secrets — created via CLI, NOT committed to Git
kubectl create secret generic serveio-secrets \
  --namespace=serveio \
  --from-literal=JWT_SECRET="$(openssl rand -base64 32)" \
  --from-literal=USERS_DATABASE_URL="postgresql://user:pass@users-db.xxx.rds.amazonaws.com:5432/users" \
  --from-literal=ORDER_DATABASE_URL="postgresql://user:pass@order-db.xxx.rds.amazonaws.com:5432/orders" \
  --from-literal=RABBITMQ_URL="amqp://admin:pass@b-xxxx.mq.us-east-1.amazonaws.com:5671"
```

### 6.3 users-service

```yaml
# k8s/users-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: users-service
  namespace: serveio
  labels:
    app: users-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: users-service
  template:
    metadata:
      labels:
        app: users-service
    spec:
      containers:
        - name: users-service
          # 1234567890 = your AWS account ID
          image: 1234567890.dkr.ecr.us-east-1.amazonaws.com/serveio/users-service:latest
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: serveio-shared-config
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: serveio-secrets
                  key: USERS_DATABASE_URL
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: serveio-secrets
                  key: JWT_SECRET
          resources:
            requests:                  # guaranteed minimum — scheduler uses this to place pods
              cpu:    "100m"           # 0.1 vCPU
              memory: "128Mi"
            limits:                    # hard cap — pod is throttled/killed if exceeded
              cpu:    "500m"
              memory: "256Mi"
          livenessProbe:
            httpGet: { path: /health/live, port: 3000 }
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet: { path: /health/ready, port: 3000 }
            initialDelaySeconds: 5
            periodSeconds: 5
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: users-service
  namespace: serveio
spec:
  selector:
    app: users-service
  ports:
    - port: 3000
      targetPort: 3000
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: users-service-hpa
  namespace: serveio
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: users-service
  minReplicas: 2
  maxReplicas: 8
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
```

### 6.4 order-service

```yaml
# k8s/order-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: serveio
spec:
  replicas: 3
  selector:
    matchLabels: { app: order-service }
  template:
    metadata:
      labels: { app: order-service }
    spec:
      containers:
        - name: order-service
          image: 1234567890.dkr.ecr.us-east-1.amazonaws.com/serveio/order-service:latest
          ports: [{ containerPort: 3000 }]
          envFrom:
            - configMapRef: { name: serveio-shared-config }
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef: { name: serveio-secrets, key: ORDER_DATABASE_URL }
            - name: RABBITMQ_URL
              valueFrom:
                secretKeyRef: { name: serveio-secrets, key: RABBITMQ_URL }
          resources:
            requests: { cpu: "150m", memory: "192Mi" }
            limits:   { cpu: "750m", memory: "384Mi" }
          livenessProbe:
            httpGet: { path: /health/live, port: 3000 }
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet: { path: /health/ready, port: 3000 }
            initialDelaySeconds: 5
            periodSeconds: 5
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: serveio
spec:
  selector: { app: order-service }
  ports: [{ port: 3000, targetPort: 3000 }]
```

### 6.5 cart-service

```yaml
# k8s/cart-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cart-service
  namespace: serveio
spec:
  replicas: 2
  selector:
    matchLabels: { app: cart-service }
  template:
    metadata:
      labels: { app: cart-service }
    spec:
      containers:
        - name: cart-service
          image: 1234567890.dkr.ecr.us-east-1.amazonaws.com/serveio/cart-service:latest
          ports: [{ containerPort: 3000 }]
          envFrom:
            - configMapRef: { name: serveio-shared-config }
          env:
            # cart-service often uses Redis for fast cart storage with TTL
            - name: REDIS_URL
              value: "redis://serveio-redis.xxxx.cache.amazonaws.com:6379"
          resources:
            requests: { cpu: "100m", memory: "128Mi" }
            limits:   { cpu: "400m", memory: "256Mi" }
          readinessProbe:
            httpGet: { path: /health/ready, port: 3000 }
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: cart-service
  namespace: serveio
spec:
  selector: { app: cart-service }
  ports: [{ port: 3000, targetPort: 3000 }]
```

### 6.6 notification-service

```yaml
# k8s/notification-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-service
  namespace: serveio
spec:
  replicas: 2
  selector:
    matchLabels: { app: notification-service }
  template:
    metadata:
      labels: { app: notification-service }
    spec:
      containers:
        - name: notification-service
          image: 1234567890.dkr.ecr.us-east-1.amazonaws.com/serveio/notification-service:latest
          ports: [{ containerPort: 4000 }]
          envFrom:
            - configMapRef: { name: serveio-shared-config }
          env:
            - name: RABBITMQ_URL
              valueFrom:
                secretKeyRef: { name: serveio-secrets, key: RABBITMQ_URL }
          resources:
            requests: { cpu: "100m", memory: "128Mi" }
            limits:   { cpu: "400m", memory: "256Mi" }
          # Notification consumers don't serve HTTP traffic the same way —
          # liveness checks the process is alive via a lightweight internal endpoint
          livenessProbe:
            httpGet: { path: /health/live, port: 4000 }
            initialDelaySeconds: 15
            periodSeconds: 15
---
apiVersion: v1
kind: Service
metadata:
  name: notification-service
  namespace: serveio
spec:
  selector: { app: notification-service }
  ports: [{ port: 4000, targetPort: 4000 }]
```

### 6.7 restaurant-service

```yaml
# k8s/restaurant-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: restaurant-service
  namespace: serveio
spec:
  replicas: 2
  selector:
    matchLabels: { app: restaurant-service }
  template:
    metadata:
      labels: { app: restaurant-service }
    spec:
      containers:
        - name: restaurant-service
          image: 1234567890.dkr.ecr.us-east-1.amazonaws.com/serveio/restaurant-service:latest
          ports: [{ containerPort: 3000 }]
          envFrom:
            - configMapRef: { name: serveio-shared-config }
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef: { name: serveio-secrets, key: RESTAURANT_DATABASE_URL }
          resources:
            requests: { cpu: "100m", memory: "128Mi" }
            limits:   { cpu: "400m", memory: "256Mi" }
          readinessProbe:
            httpGet: { path: /health/ready, port: 3000 }
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: restaurant-service
  namespace: serveio
spec:
  selector: { app: restaurant-service }
  ports: [{ port: 3000, targetPort: 3000 }]
```

### 6.8 serveio-frontend

```yaml
# k8s/serveio-frontend/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: serveio-frontend
  namespace: serveio
spec:
  replicas: 2
  selector:
    matchLabels: { app: serveio-frontend }
  template:
    metadata:
      labels: { app: serveio-frontend }
    spec:
      containers:
        - name: serveio-frontend
          image: 1234567890.dkr.ecr.us-east-1.amazonaws.com/serveio/frontend:latest
          ports: [{ containerPort: 80 }]    # served via nginx in the container
          resources:
            requests: { cpu: "50m", memory: "64Mi" }
            limits:   { cpu: "200m", memory: "128Mi" }
          readinessProbe:
            httpGet: { path: /, port: 80 }
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: serveio-frontend
  namespace: serveio
spec:
  selector: { app: serveio-frontend }
  ports: [{ port: 80, targetPort: 80 }]
```

### 6.9 Ingress — routing all services

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: serveio-ingress
  namespace: serveio
  annotations:
    kubernetes.io/ingress.class:                  alb
    alb.ingress.kubernetes.io/scheme:              internet-facing
    alb.ingress.kubernetes.io/target-type:         ip
    alb.ingress.kubernetes.io/listen-ports:        '[{"HTTPS":443}, {"HTTP":80}]'
    alb.ingress.kubernetes.io/certificate-arn:     arn:aws:acm:us-east-1:1234567890:certificate/xxxx
    alb.ingress.kubernetes.io/ssl-redirect:        '443'
    alb.ingress.kubernetes.io/healthcheck-path:    /health/ready
spec:
  rules:
    - host: api.serveio.com
      http:
        paths:
          - path: /api/users
            pathType: Prefix
            backend: { service: { name: users-service, port: { number: 3000 } } }
          - path: /api/orders
            pathType: Prefix
            backend: { service: { name: order-service, port: { number: 3000 } } }
          - path: /api/cart
            pathType: Prefix
            backend: { service: { name: cart-service, port: { number: 3000 } } }
          - path: /api/notifications
            pathType: Prefix
            backend: { service: { name: notification-service, port: { number: 4000 } } }
          - path: /api/restaurants
            pathType: Prefix
            backend: { service: { name: restaurant-service, port: { number: 3000 } } }
    - host: serveio.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: serveio-frontend, port: { number: 80 } } }
```

---

## 7. Setting Up AWS Infrastructure

### 7.1 Prerequisites

```bash
# Install required CLI tools
brew install awscli eksctl kubectl helm     # macOS
# or for Linux: use the official install scripts for each tool

# Configure AWS credentials
aws configure
# AWS Access Key ID, Secret Access Key, region (e.g. us-east-1)

# Verify
aws sts get-caller-identity
```

### 7.2 Creating the EKS Cluster

```bash
# eksctl is the simplest way to stand up a production-ready EKS cluster
eksctl create cluster \
  --name serveio-cluster \
  --region us-east-1 \
  --version 1.30 \
  --nodegroup-name serveio-workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 6 \
  --managed
# This takes ~15 minutes — provisions VPC, subnets, IAM roles, and EC2 nodes

# Once done, kubectl is configured automatically. Verify:
kubectl get nodes
# NAME                          STATUS   ROLES    AGE   VERSION
# ip-192-168-1-1.ec2.internal   Ready    <none>   2m    v1.30.0
```

**Or with a config file (recommended for reproducibility):**

```yaml
# eks-cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: serveio-cluster
  region: us-east-1
  version: "1.30"

managedNodeGroups:
  - name: serveio-workers
    instanceType: t3.medium
    minSize: 2
    maxSize: 6
    desiredCapacity: 3
    volumeSize: 30
    labels:
      role: worker
    iam:
      withAddonPolicies:
        autoScaler: true
        cloudWatch: true
        ebs: true

addons:
  - name: vpc-cni
  - name: coredns
  - name: kube-proxy
  - name: aws-ebs-csi-driver
```

```bash
eksctl create cluster -f eks-cluster.yaml
```

### 7.3 Setting Up ECR (Container Registry)

```bash
# Create a repository for each service
for service in users-service order-service cart-service notification-service restaurant-service frontend; do
  aws ecr create-repository --repository-name serveio/$service --region us-east-1
done

# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 1234567890.dkr.ecr.us-east-1.amazonaws.com

# Build, tag, and push each service
cd order-service
docker build -t serveio/order-service:latest .
docker tag serveio/order-service:latest 1234567890.dkr.ecr.us-east-1.amazonaws.com/serveio/order-service:latest
docker push 1234567890.dkr.ecr.us-east-1.amazonaws.com/serveio/order-service:latest
```

### 7.4 Installing the AWS Load Balancer Controller

This is what makes the `Ingress` resource actually create a real AWS ALB.

```bash
# 1. Create an IAM policy for the controller
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.0/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# 2. Create a service account with that policy attached (IRSA)
eksctl create iamserviceaccount \
  --cluster=serveio-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::1234567890:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# 3. Install the controller via Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=serveio-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Verify
kubectl get deployment -n kube-system aws-load-balancer-controller
```

### 7.5 RDS, ElastiCache, and Amazon MQ

**Why use managed AWS services instead of running Postgres/Redis/RabbitMQ in pods?** Stateful workloads in Kubernetes are operationally complex (backups, failover, patching). Managed services handle this for you — let Kubernetes focus on your stateless Node.js services.

```bash
# RDS Postgres for order-service
aws rds create-db-instance \
  --db-instance-identifier serveio-order-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 16 \
  --master-username serveio_admin \
  --master-user-password "ChangeMe123!" \
  --allocated-storage 20 \
  --vpc-security-group-ids sg-xxxxxxxx \
  --db-subnet-group-name serveio-db-subnet-group

# ElastiCache Redis for cart-service
aws elasticache create-cache-cluster \
  --cache-cluster-id serveio-cart-redis \
  --engine redis \
  --cache-node-type cache.t3.micro \
  --num-cache-nodes 1 \
  --security-group-ids sg-xxxxxxxx

# Amazon MQ (managed RabbitMQ) for notification-service event bus
aws mq create-broker \
  --broker-name serveio-rabbitmq \
  --engine-type RABBITMQ \
  --engine-version 3.13 \
  --host-instance-type mq.t3.micro \
  --deployment-mode SINGLE_INSTANCE \
  --users '[{"Username":"admin","Password":"ChangeMe123!"}]' \
  --publicly-accessible false
```

> **Networking note:** Your EKS nodes and these managed services must be in the same VPC (or peered VPCs) with security groups that allow traffic on the relevant ports (5432 for Postgres, 6379 for Redis, 5671 for Amazon MQ over TLS).

---

## 8. CI/CD — GitHub Actions to EKS

Based on the `.github/workflows` folder visible in the repo, here's a production pipeline that builds and deploys on every push.

```yaml
# .github/workflows/deploy-order-service.yml
name: Deploy order-service

on:
  push:
    branches: [main]
    paths:
      - 'order-service/**'          # only trigger when this service changes

env:
  AWS_REGION:    us-east-1
  ECR_REPO:      serveio/order-service
  EKS_CLUSTER:   serveio-cluster
  NAMESPACE:     serveio

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id:     ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region:             ${{ env.AWS_REGION }}

      - name: Login to ECR
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image
        working-directory: ./order-service
        env:
          REGISTRY: ${{ steps.ecr-login.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $REGISTRY/$ECR_REPO:$IMAGE_TAG -t $REGISTRY/$ECR_REPO:latest .
          docker push $REGISTRY/$ECR_REPO:$IMAGE_TAG
          docker push $REGISTRY/$ECR_REPO:latest

      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name $EKS_CLUSTER --region $AWS_REGION

      - name: Deploy to EKS (rolling update)
        env:
          REGISTRY: ${{ steps.ecr-login.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          kubectl set image deployment/order-service \
            order-service=$REGISTRY/$ECR_REPO:$IMAGE_TAG \
            -n $NAMESPACE

          # Wait for the rollout to finish — fails the pipeline if it doesn't
          # complete within the timeout (catches broken deploys automatically)
          kubectl rollout status deployment/order-service -n $NAMESPACE --timeout=180s

      - name: Rollback on failure
        if: failure()
        run: |
          echo "Deployment failed — rolling back"
          kubectl rollout undo deployment/order-service -n $NAMESPACE
```

**Run database migrations before the rollout (as a Job):**

```yaml
      - name: Run DB migration
        env:
          REGISTRY: ${{ steps.ecr-login.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          # Delete any previous migration job, then run a fresh one
          kubectl delete job order-service-migrate -n $NAMESPACE --ignore-not-found
          kubectl create job order-service-migrate \
            --image=$REGISTRY/$ECR_REPO:$IMAGE_TAG \
            -n $NAMESPACE \
            -- npm run migrate
          kubectl wait --for=condition=complete job/order-service-migrate -n $NAMESPACE --timeout=120s
```

---

## 9. Helm — Packaging Your Services

Writing raw YAML for 6 services × (Deployment + Service + HPA) gets repetitive fast. **Helm** templates this so you maintain one chart and override values per service.

```
helm/
└── node-service/                  # generic reusable chart for ANY Node.js service
    ├── Chart.yaml
    ├── values.yaml                # defaults
    ├── values-order-service.yaml  # override for order-service
    ├── values-cart-service.yaml
    └── templates/
        ├── deployment.yaml
        ├── service.yaml
        └── hpa.yaml
```

```yaml
# helm/node-service/values.yaml
name: my-service
image:
  repository: 1234567890.dkr.ecr.us-east-1.amazonaws.com/serveio/my-service
  tag: latest
replicaCount: 2
port: 3000
resources:
  requests: { cpu: "100m", memory: "128Mi" }
  limits:   { cpu: "400m", memory: "256Mi" }
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 8
  targetCPU: 70
env: {}
```

```yaml
# helm/node-service/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.name }}
  namespace: {{ .Release.Namespace }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels: { app: {{ .Values.name }} }
  template:
    metadata:
      labels: { app: {{ .Values.name }} }
    spec:
      containers:
        - name: {{ .Values.name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports: [{ containerPort: {{ .Values.port }} }]
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          envFrom:
            - configMapRef: { name: serveio-shared-config }
          readinessProbe:
            httpGet: { path: /health/ready, port: {{ .Values.port }} }
            initialDelaySeconds: 5
            periodSeconds: 5
```

```yaml
# helm/node-service/values-order-service.yaml
name: order-service
image:
  tag: "1.1.0"
replicaCount: 3
port: 3000
autoscaling:
  maxReplicas: 10
```

```bash
# Deploy order-service using the shared chart + its overrides
helm upgrade --install order-service ./helm/node-service \
  -f ./helm/node-service/values-order-service.yaml \
  -n serveio

# Deploy cart-service using the SAME chart + different overrides
helm upgrade --install cart-service ./helm/node-service \
  -f ./helm/node-service/values-cart-service.yaml \
  -n serveio
```

---

## 10. Observability — Logs, Metrics, Tracing

```bash
# View live logs from a specific pod
kubectl logs -f deployment/order-service -n serveio

# View logs from ALL pods of a deployment (requires kubectl 1.28+ or a label selector)
kubectl logs -f -l app=order-service -n serveio --max-log-requests=10

# Stream logs with a quick grep for errors
kubectl logs -f deployment/order-service -n serveio | grep -i error
```

**Production setup: CloudWatch Container Insights**

```bash
# Install the CloudWatch agent + Fluent Bit for log/metric shipping
aws eks create-addon \
  --cluster-name serveio-cluster \
  --addon-name amazon-cloudwatch-observability
```

```typescript
// Structured JSON logging in Node.js — makes CloudWatch Logs Insights queries useful
import winston from 'winston';

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),   // JSON logs are queryable in CloudWatch
  defaultMeta: { service: 'order-service' },
  transports: [new winston.transports.Console()],
});

logger.info('Order created', { orderId: order.id, userId: order.userId, amount: order.total });
// CloudWatch Logs Insights query:
// fields @timestamp, orderId, userId, amount
// | filter service = "order-service"
// | sort @timestamp desc
```

**Distributed tracing** (recommended: AWS X-Ray or OpenTelemetry):

```typescript
// Basic OpenTelemetry setup for tracing requests across services
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({ url: 'http://otel-collector:4318/v1/traces' }),
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();
// Now every HTTP call between order-service → users-service → notification-service
// is automatically traced with a shared trace ID, visible in AWS X-Ray.
```

```bash
kubectl top pods -n serveio          # quick CPU/memory snapshot per pod
kubectl top nodes                    # cluster-wide resource usage
```

---

## 11. Scaling Strategies

| Strategy | Mechanism | When to Use |
|---|---|---|
| **Horizontal Pod Autoscaling** | HPA watches CPU/memory, adds pod replicas | Traffic-driven services (order-service, cart-service) |
| **Cluster Autoscaler** | Adds/removes EC2 nodes when pods can't be scheduled | Whole-cluster capacity, paired with HPA |
| **Vertical Pod Autoscaler** | Adjusts CPU/memory requests automatically | Right-sizing resource requests over time |
| **KEDA (event-driven autoscaling)** | Scale based on queue depth, not just CPU | notification-service scaling with RabbitMQ queue length |

```bash
# Cluster Autoscaler — scales EC2 nodes, not pods
eksctl create iamserviceaccount \
  --cluster=serveio-cluster \
  --namespace=kube-system \
  --name=cluster-autoscaler \
  --attach-policy-arn=arn:aws:iam::aws:policy/AutoScalingFullAccess \
  --approve

helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  -n kube-system \
  --set autoDiscovery.clusterName=serveio-cluster \
  --set awsRegion=us-east-1
```

```yaml
# KEDA ScaledObject — scale notification-service based on RabbitMQ queue depth
# (instead of just CPU, which doesn't reflect a backed-up queue)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: notification-service-scaler
  namespace: serveio
spec:
  scaleTargetRef:
    name: notification-service
  minReplicaCount: 1
  maxReplicaCount: 15
  triggers:
    - type: rabbitmq
      metadata:
        queueName: notification.inbox
        host: amqp://admin:pass@b-xxxx.mq.us-east-1.amazonaws.com:5672
        value: "20"          # scale up when avg 20 messages per pod are queued
```

---

## 12. Security Checklist

| # | Check | Implementation |
|---|---|---|
| 1 | **Run containers as non-root** | `USER nodeuser` in Dockerfile + `securityContext.runAsNonRoot: true` |
| 2 | **Use AWS Secrets Manager, not raw K8s Secrets** | External Secrets Operator syncs AWS secrets into K8s automatically |
| 3 | **Restrict pod-to-pod traffic** | NetworkPolicies — only allow order-service to talk to users-service, not everything |
| 4 | **Least-privilege IAM roles** | IRSA (IAM Roles for Service Accounts) — each pod gets only the AWS permissions it needs |
| 5 | **Set resource limits on every container** | Prevents one misbehaving pod from starving the node |
| 6 | **Enable encryption at rest for Secrets** | `aws eks` cluster created with `--encryption-config` using a KMS key |
| 7 | **Scan images for vulnerabilities** | ECR image scanning (`aws ecr put-image-scanning-configuration`) |
| 8 | **Use private ECR + private subnets for nodes** | Nodes shouldn't be internet-facing; only the ALB is |
| 9 | **Rotate JWT secrets and DB credentials** | AWS Secrets Manager rotation schedules |
| 10 | **Limit `kubectl` access with RBAC** | Role/RoleBinding per team — don't give everyone cluster-admin |

```yaml
# securityContext example — add to every Deployment
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      containers:
        - name: order-service
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
```

```yaml
# NetworkPolicy — order-service can ONLY be called by users-service and the ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-policy
  namespace: serveio
spec:
  podSelector:
    matchLabels: { app: order-service }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: users-service } }
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } } # allow ALB
      ports: [{ protocol: TCP, port: 3000 }]
```

**External Secrets Operator (recommended over raw K8s Secrets):**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: order-service-secrets
  namespace: serveio
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: serveio-secrets       # creates a regular K8s Secret with this name
  data:
    - secretKey: ORDER_DATABASE_URL
      remoteRef:
        key: serveio/order-service/database-url
```

---

## 13. Common Pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| **No readiness probe** | Traffic sent to pods still booting → 502 errors during deploys | Add `readinessProbe` checking real DB connectivity |
| **No graceful shutdown handling** | Dropped requests during every rolling update | Handle `SIGTERM`, close server, drain connections (Section 5) |
| **No resource limits** | One pod OOMs and takes down the node | Set `resources.requests` and `resources.limits` on every container |
| **Using `latest` tag in production** | Can't roll back to a known-good version | Tag images with git SHA or semver; never deploy `:latest` |
| **Secrets committed to Git** | Credential leak | Use `kubectl create secret` from CLI, or External Secrets Operator |
| **ConfigMap/Secret changes don't restart pods** | Old config still in use after `kubectl apply` | Use a checksum annotation, or `kubectl rollout restart deployment/x` |
| **Single replica for critical services** | Any pod restart = downtime | `minReplicas: 2` minimum for anything user-facing |
| **CPU limits set too low** | Node.js event loop stalls under load (throttled) | Profile actual usage; avoid over-restrictive `limits.cpu` |
| **Forgetting `maxUnavailable: 0`** | Rolling updates briefly reduce capacity below desired | Set `maxUnavailable: 0` for zero-downtime guarantees on critical paths |
| **No Ingress health check path configured** | ALB marks healthy pods as unhealthy | Set `alb.ingress.kubernetes.io/healthcheck-path` to your real readiness endpoint |
| **Hardcoded service URLs** | Breaks when moving namespaces/environments | Always use ConfigMaps for inter-service URLs (cluster DNS names) |
| **No Pod Disruption Budget** | Cluster autoscaler / node upgrades kill all replicas at once | Add a `PodDisruptionBudget` to guarantee minimum availability |

```yaml
# PodDisruptionBudget — ensures at least 2 order-service pods stay up
# during voluntary disruptions (node drains, cluster upgrades)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: serveio
spec:
  minAvailable: 2
  selector:
    matchLabels: { app: order-service }
```

---

## Quick Command Reference

```bash
# Cluster
kubectl get nodes                                    # list cluster nodes
kubectl cluster-info                                  # control plane endpoints

# Deployments
kubectl get deployments -n serveio
kubectl describe deployment order-service -n serveio
kubectl scale deployment order-service --replicas=5 -n serveio
kubectl rollout status deployment/order-service -n serveio
kubectl rollout undo deployment/order-service -n serveio
kubectl rollout history deployment/order-service -n serveio

# Pods
kubectl get pods -n serveio -o wide
kubectl describe pod <pod-name> -n serveio
kubectl logs -f <pod-name> -n serveio
kubectl exec -it <pod-name> -n serveio -- /bin/sh

# Services & Ingress
kubectl get svc -n serveio
kubectl get ingress -n serveio
kubectl describe ingress serveio-ingress -n serveio

# Debugging
kubectl get events -n serveio --sort-by='.lastTimestamp'
kubectl top pods -n serveio
kubectl get hpa -n serveio
```

---

*Generated for Node.js 20 · Kubernetes 1.30 · AWS EKS · Based on a microservices architecture (users, orders, cart, notifications, restaurants, frontend)*