# 🔐 Secure Containers & Kubernetes Hardening Guide
## Docker Multi-Stage Builds, Non-Root Containers, Pod Security, Debugging & Stateful Databases

---

## 📌 Purpose of This Document

This README is a **complete production-grade guide** for:

- Building **secure Docker images** using **multi-stage builds**
- Running containers as **non-root users**
- Hardening **normal application Deployments**
- Hardening **databases using StatefulSets**
- Enforcing security at **Kubernetes Pod & Namespace level**
- Debugging **production workloads without root**
- Understanding **when `kubectl debug` gets root and when it does NOT**

This document reflects **real DevOps / DevSecOps practices used in production environments**.

---

## 🧠 Core Security Model (Very Important)

Docker Image Security  →  Kubernetes Enforcement  →  Observability & Debugging

Dockerfile → Secure by default  
Kubernetes → Enforce runtime security  
Observability → Debug without root  

Kubernetes cannot fix insecure images.  
Secure images still need Kubernetes enforcement.

---

## 🐳 Docker Security: Multi-Stage Builds

### Why Multi-Stage Builds?

- Build tools are removed from runtime
- Smaller images
- Reduced attack surface
- Faster startup
- Faster vulnerability scans

---

## 🔐 Golden Rules for Secure Dockerfiles

1. Build stage may run as root
2. Runtime stage must run as non-root
3. Explicitly create a runtime user
4. Fix ownership of writable paths
5. Use ports > 1024
6. USER must be the last instruction

---

## ✅ Secure Multi-Stage Dockerfile Example (Node.js)

```dockerfile
############################
# Build Stage
############################
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build


############################
# Runtime Stage
############################
FROM node:20-alpine

RUN addgroup -g 1001 appgroup  && adduser -D -u 1001 -G appgroup appuser

WORKDIR /app

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 3000
CMD ["node","dist/index.js"]
```

---

## 🔍 Explanation of the Critical User Creation Line

```dockerfile
RUN addgroup -g 1001 appgroup && adduser -D -u 1001 -G appgroup appuser
```

### What this does

- Creates a non-root Linux group
- Creates a non-root Linux user
- Assigns fixed UID/GID = 1001
- Ensures predictable permissions
- Works cleanly with Kubernetes fsGroup

### Why UID/GID = 1001?

- UID 0 is root and insecure
- Fixed IDs avoid permission mismatches
- Kubernetes volumes work reliably
- Security scanners pass cleanly

---

# ☸️ Kubernetes Runtime Security (Pod-Level)

Dockerfile security alone is not enough.

---

## 🔐 Mandatory Pod Security Context

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1001
  runAsGroup: 1001
  fsGroup: 1001
```

### What each field does

- runAsNonRoot → Pod fails if container tries to run as root
- runAsUser → Forces UID
- runAsGroup → Forces GID
- fsGroup → Fixes volume permissions after mount

---

## 🔐 Container-Level Hardening (ALL WORKLOADS)

```yaml
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

- Blocks sudo and setuid
- Prevents malware persistence
- Blocks kernel-level abuse

---

## 📦 Writable Paths with Read-Only Root FS

```yaml
volumeMounts:
- name: tmp
  mountPath: /tmp
- name: storage
  mountPath: /app/storage

volumes:
- name: tmp
  emptyDir: {}
- name: storage
  emptyDir: {}
```

---

# 🚀 Normal Application Deployment Hardening

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 2
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001

      containers:
      - name: app
        image: my-app:latest
        ports:
        - containerPort: 3000
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]

        volumeMounts:
        - name: tmp
          mountPath: /tmp

      volumes:
      - name: tmp
        emptyDir: {}
```

---

# 🗄️ Database Hardening with StatefulSets

Databases are stateful and high-risk.

---

## 🔐 Database Hardening Rules

- Never run DB as root
- Always use StatefulSet
- Use PVCs (never hostPath)
- Configure fsGroup
- Drop capabilities
- Use Kubernetes Secrets
- Restrict network access

---

## ✅ Secure PostgreSQL StatefulSet Example

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999

      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data

  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

---

## 🔐 Database Network Isolation

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-only-backend
spec:
  podSelector:
    matchLabels:
      app: postgres
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
```

---

# 🛠️ Debugging Production Systems (NO ROOT)

## Why Root Is Not Needed

- Root increases blast radius
- Breaks compliance
- Hides design flaws
- Kubernetes provides safer debugging tools

---

## 🔍 kubectl debug — Ephemeral Containers

```bash
kubectl debug pod-name -it --image=busybox --target=app
```

### What this does

- Injects a temporary debug container
- Does NOT restart the app
- Shares network, PID, and volumes
- Does NOT modify the app image
- Automatically removed after exit

---

## 🚨 When kubectl debug Gets Root

- Root is only inside the debug container
- Never inside the app container
- Only if namespace allows it

---

## ❌ When Root Is Blocked

- Pod Security: restricted enforced
- runAsNonRoot applied to ephemeral containers

---

## 🔁 Safe Debug Alternatives

```bash
kubectl logs pod-name
kubectl port-forward pod-name 8080:3000
```

---

# 🚨 Common Failure Scenarios

| Issue | Root Cause | Fix |
|----|----|----|
| Permission denied | fsGroup missing | Add fsGroup |
| CrashLoopBackOff | Read-only FS | Mount writable volume |
| Pod won’t start | Image runs as root | Fix Dockerfile |
| Works locally | UID mismatch | Use fsGroup |

---

# 📋 Production Security Checklist

- Multi-stage Docker builds
- Non-root runtime user
- Fixed UID/GID
- chown writable paths
- Pod securityContext enforced
- fsGroup configured
- Capabilities dropped
- Read-only root filesystem
- Secure Deployments
- Secure StatefulSets
- NetworkPolicies applied

---

## 🎯 Interview-Ready Summary

We build secure non-root images using multi-stage Docker builds, enforce runtime security using Kubernetes pod and namespace policies, harden both stateless Deployments and StatefulSet databases, and debug production safely using logs, port-forwarding, and ephemeral containers without relying on root access.

---

## 🚀 Final Note

This setup is production-ready, security-auditable, cloud-native, and enterprise-approved.

