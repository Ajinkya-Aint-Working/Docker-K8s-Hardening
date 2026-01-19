# 🔐 Secure Containers & Kubernetes Hardening Guide
## Docker Multi-Stage Builds, Non-Root Containers, Pod Security & Database StatefulSets

---

## 📌 Purpose of This Document

This README provides a **production-grade guide** for:

- Building **secure Docker images** using multi-stage builds
- Running containers as **non-root users**
- Enforcing security at the **Kubernetes Pod and Namespace level**
- Hardening **database workloads using StatefulSets**
- Debugging production systems **without root access**, including `kubectl debug`

This document reflects **real-world DevOps & DevSecOps practices**.

---

## 🧠 Core Security Model

Docker Image Security → Kubernetes Enforcement → Observability

- **Dockerfile**: Secure by default
- **Kubernetes**: Enforces runtime rules
- **Observability**: Debug without root

⚠ Kubernetes cannot fix insecure images  
⚠ Secure images still require Kubernetes enforcement  
✅ Both are mandatory

---

## 🐳 Docker: Secure Multi-Stage Builds

### Why Multi-Stage Builds?

- Removes build tools from runtime
- Smaller images
- Reduced attack surface
- Faster security scans

---

### 🔐 Golden Rules for Secure Dockerfiles

1. Build stage may run as root
2. Runtime stage must run as non-root
3. Explicitly create a runtime user
4. Fix ownership of writable paths
5. Use ports > 1024
6. `USER` must be the last instruction

---

## ✅ Secure Multi-Stage Dockerfile Example

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:20-alpine AS runtime
RUN addgroup -g 1001 appgroup && adduser -D -u 1001 -G appgroup appuser
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

## 🔍 Explanation of Critical User Creation Line

```dockerfile
RUN addgroup -g 1001 appgroup && adduser -D -u 1001 -G appgroup appuser
```

- Creates a non-root Linux user and group
- Uses fixed UID/GID (1001) for predictable permissions
- Ensures Kubernetes `fsGroup` compatibility
- Prevents accidental root execution

UID/GID 1001 is a safe, widely accepted production standard.

---

## ☸️ Kubernetes Pod-Level Security Enforcement

### Mandatory Pod Security Context

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1001
  runAsGroup: 1001
  fsGroup: 1001
```

### What Each Field Does

- `runAsNonRoot`: Pod fails if container tries to run as root
- `runAsUser`: Forces UID inside container
- `runAsGroup`: Ensures group permission consistency
- `fsGroup`: Fixes permissions on mounted volumes (CRITICAL)

---

## 🔐 Container-Level Hardening

```yaml
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

- Blocks privilege escalation
- Prevents filesystem tampering
- Drops unnecessary Linux capabilities

---

## 📦 Writable Paths with Read-Only Root Filesystem

```yaml
volumeMounts:
- name: storage
  mountPath: /app/storage
- name: tmp
  mountPath: /tmp

volumes:
- name: storage
  emptyDir: {}
- name: tmp
  emptyDir: {}
```

---

## 🗄️ Hardening Databases Using StatefulSets

### Why Databases Need Extra Hardening

- Stateful data
- Persistent volumes
- High blast radius if compromised

---

## 🔐 Database Hardening Rules

- Never run database containers as root
- Always use StatefulSets (not Deployments)
- Use PersistentVolumeClaims (PVCs)
- Configure `fsGroup`
- Drop Linux capabilities
- Restrict network access
- Store credentials in Kubernetes Secrets

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

## 🔐 Network Policy for Databases

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-only
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

## 🛠️ Debugging Without Root Access (CRITICAL SECTION)

### Why Root Is Not Needed in Production

- Root access increases blast radius
- Modern systems rely on observability
- Kubernetes provides safer debugging mechanisms

---

## 🔍 `kubectl debug` — EPHEMERAL DEBUGGING EXPLAINED

### Command

```bash
kubectl debug pod-name -it --image=busybox --target=app
```

### What This Command Does

- Injects a **temporary (ephemeral) container** into an existing pod
- Does **not restart** the running application container
- Shares **network namespace**, **PID namespace**, and **volumes**
- Does **not modify** the original container image
- Automatically removed when session ends

---

### Flag-by-Flag Explanation

| Flag | Meaning |
|----|----|
| `kubectl debug` | Start debug session |
| `pod-name` | Target pod |
| `-it` | Interactive terminal |
| `--image=busybox` | Lightweight debug image |
| `--target=app` | Attach to app container namespaces |

---

### Why `--target=app` Is Important

- Allows inspection of:
  - Open ports
  - Network connectivity
  - Mounted volumes
- Without modifying production image
- Without requiring root

---

### What You Can Safely Do

```bash
ps aux
ls -la /app
wget http://localhost:3000/health
netstat -tulpn
```

### What You CANNOT Do

- Become root in app container
- Modify production binaries
- Persist changes after exit

---

### Why This Is Production-Safe

- No image rebuild
- No pod restart
- Auditable via Kubernetes events
- Recommended by Kubernetes SIG Security

---

## 🔁 Alternative Debug Methods

### Port Forwarding (Preferred)

```bash
kubectl port-forward pod-name 8080:3000
curl http://localhost:8080/health
```

### Logs

```bash
kubectl logs pod-name
kubectl logs pod-name --previous
```

---

## 🚨 Common Failure Scenarios

| Issue | Cause | Fix |
|-----|------|-----|
| Permission denied | fsGroup missing | Add fsGroup |
| CrashLoopBackOff | RO filesystem | Mount writable volume |
| Pod won’t start | Root image | Fix Dockerfile |
| Works locally, fails in K8s | UID mismatch | fsGroup / emptyDir |

---

## 📋 Production Security Checklist

- Multi-stage Docker builds
- Non-root runtime users
- Fixed UID/GID
- Writable paths explicitly owned
- Pod-level securityContext enforced
- fsGroup configured
- Capabilities dropped
- Databases in StatefulSets
- Secrets for credentials
- NetworkPolicies applied

---

## 🎯 Interview-Ready Summary

“We use multi-stage Docker builds with non-root users, enforce Kubernetes pod-level security, harden StatefulSet databases, and debug production systems safely using logs, port-forwarding, and ephemeral debug containers without root access.”

---

## 🚀 Final Note

This setup is **production-ready**, **security-auditable**, and **used by modern cloud-native teams**.

Mastering this early gives a **significant DevOps career advantage**.
