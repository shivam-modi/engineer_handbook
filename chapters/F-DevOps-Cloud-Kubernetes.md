# F. DevOps, Cloud & Kubernetes (Principal Engineer Depth)

This is one of the most practical chapters:  
**this is exactly what SDE3/Staff/Platform/Backend Infrastructure interviews check.**

---

# Table of Contents

1. Linux Internals for Backend Engineers  
2. Processes, Threads, Syscalls, Scheduling  
3. Namespaces & Cgroups (Containers Foundation)  
4. Docker Internals (image layers, overlayfs, container runtime)  
5. Container Networking  
6. Kubernetes Deep Architecture  
7. Nodes, Pods, Kubelet, Runtime  
8. Control Plane Internals  
9. Kubernetes Networking (CNI)  
10. Services, DNS, Ingress, Load Balancers  
11. Horizontal Pod Autoscaling  
12. Service Mesh (Istio/Envoy)  
13. Cloud Architecture Designs (AWS/GCP)  
14. CI/CD Patterns (Blue/Green, Canary, Shadow)  
15. Infrastructure-as-Code  
16. Production Safety & Deployment Patterns  

---

# 1. LINUX INTERNALS (DEEP)

Backend & cloud systems run on **Linux**, so you must understand:

- processes  
- threads  
- CPU scheduling  
- virtual memory  
- file descriptors  
- networking stack  
- cgroups / namespaces  

---

## 1.1 Process vs Thread

### Process:
- Owns memory (MMU maps pages)
- Owns file descriptors
- Heavy to create (copy-on-write helps)

### Thread:
- Shares memory with other threads  
- Cheaper to create  
- Represented by **task_struct** in kernel  

---

## 1.2 System Calls (important)

Every system-level operation → syscall:

```
read()
write()
accept()
sendfile()
open()
futex()
epoll_wait()
```

Know **epoll**:  
foundation of high-performance servers (Nginx, Netty, Node.js).

---

## 1.3 CFS — Completely Fair Scheduler

Linux schedules threads via:

- vruntime  
- fair-share scheduling  
- prioritizing short tasks  

---

## 1.4 File Descriptors & Limits

Every socket = file descriptor.

Check limits:

```bash
ulimit -n
```

If FD exhausted → service cannot accept connections → 502s.

---

# 2. CGROUPS & NAMESPACES (CONTAINERS FOUNDATION)

Containers = **not VMs**, they are just:

✓ Namespaces (isolation)  
✓ Cgroups (resource limits)  
✓ Union file systems (overlayfs)  

---

## 2.1 Namespaces (isolation)

```
PID Namespace      → isolation of processes
Network Namespace  → virtual network
Mount Namespace    → different mounts
User Namespace     → UID/GID mapping
IPC Namespace      → message queues
UTS Namespace      → hostname isolation
```

---

## 2.2 Cgroups (resource control)

Control:

- CPU shares
- Memory limits
- I/O throttling

Example:

```bash
docker run --memory=512m --cpus=1 nginx
```

Memory cgroup → OOM killer inside container.

---

# 3. DOCKER INTERNALS (VERY DEEP)

## 3.1 Docker Architecture

```
Docker CLI
   ↓
Docker Engine (API)
   ↓
Containerd
   ↓
runc (OCI runtime)
   ↓
Linux cgroups + namespaces
```

---

## 3.2 Image Layers (Union FS)

Every Docker image layer = immutable:

```
FROM ubuntu
 → Layer 1
RUN apt update
 → Layer 2
COPY . .
 → Layer 3
```

Delete files?  
**Still stored in lower layers**, increasing size.

---

## 3.3 OverlayFS

Union filesystem:

```
LowerDir  (base image layers)
UpperDir  (container writes)
MergedDir (combined)
```

Writes go to `UpperDir`.

---

## 3.4 Container Networking (CNM)

Docker has:

- bridge networks  
- host network  
- macvlan  

Flow:

```
Container veth0
    ↕
veth pair
    ↕
Linux Bridge (docker0)
    ↕
iptables NAT
    ↕
eth0 (host)
```

---

# 4. KUBERNETES (EXTREMELY DEEP)

K8s is a **declarative control-plane** orchestrating containers.

---

# 4.1 High-Level Architecture Diagram

```
               ┌─────────────────────────┐
               │      Control Plane      │
               ├─────────────────────────┤
               │  API Server (front door)│
               │  Scheduler              │
               │  Controller Manager     │
               │  etcd (cluster state)   │
               └───────────┬─────────────┘
                           │
          ┌────────────────▼────────────────┐
          │              Nodes              │
          ├─────────────────────────────────┤
          │ Kubelet     │ runs pods        │
          │ Kube-Proxy  │ networking rules │
          │ ContainerD  │ runtime          │
          └─────────────────────────────────┘
```

---

# 5. CONTROL PLANE INTERNALS

### 1. API Server
- Accepts all requests  
- Validates & stores objects in `etcd`  
- Stateless  

### 2. Scheduler
- Decides which node runs pod  
- Based on:
  - CPU/memory  
  - taints/tolerations  
  - affinity rules  
  - resource availability  

### 3. Controller Manager
Reconciles desired state.

Examples:
- ReplicaSet controller  
- Deployment controller  
- Node controller  

---

# 6. NODE COMPONENTS

### Kubelet
- Watches API server  
- Ensures pods are running  
- Talks to container runtime  

### Kube-Proxy
- Programs iptables/IPVS rules  
- Implements Kubernetes Services  

### Container Runtime
- containerd
- runc (OCI)

---

# 7. PODS (CORE ABSTRACTION)

Pod = **smallest deployable unit**.

Contains:
- 1 or more containers  
- shared network namespace  
- shared volume mounts  

Sidecar pattern:

```
[ App Container ]
[ Sidecar (Envoy/Fluentbit/Init) ]
```

---

# 8. KUBERNETES NETWORKING (DEEP)

K8s networking rules:

1) Every Pod gets its own IP  
2) Pods can talk to each other without NAT  
3) Services provide stable virtual IP  

CNI (Container Network Interface) implements:

- Calico  
- Cilium (BPF-based)  
- Flannel  
- Weave  

---

# 9. SERVICE TYPES

### ClusterIP (default)
Internal only.

### NodePort
Expose on each node IP.

### LoadBalancer
Cloud LB external access.

### Headless Service
`clusterIP: None`, used for:
- StatefulSets  
- gRPC load balancing  

---

# 10. DEPLOYMENTS

Deployment ensures:

```
desired replicas = actual replicas
```

Supports:
- rolling updates  
- rollbacks  
- revision history  

---

# 11. HORIZONTAL POD AUTOSCALING (HPA)

Scales based on:

- CPU usage  
- Memory  
- Custom metrics (request rate, queue length)  

Uses control theory:

```
Target Utilization = (current / target)
```

---

# 12. SERVICE MESH (ENVOY / ISTIO)

Service mesh provides:

- mTLS  
- retries  
- circuit breaking  
- rate limiting  
- observability  
- traffic splitting  

Sidecar architecture:

```
Pod:
 ├── App Container
 └── Envoy Sidecar (proxy)
```

Envoy intercepts all traffic.

---

# 13. CLOUD ARCHITECTURE (AWS/GCP)

## 13.1 Common AWS building blocks

- EC2 (compute)  
- ASG (scaling)  
- ALB/NLB (load balancers)  
- S3 (object storage)  
- RDS/Aurora (databases)  
- DynamoDB (NoSQL)  
- SQS/SNS (messaging)  
- VPC subnets, NAT gateways  

---

## 13.2 VPC Layout Diagram

```
VPC
├── Public Subnet (ALB, NAT)
└── Private Subnet (EKS Nodes, DB)
```

---

# 14. CI/CD PATTERNS (DEEP)

### 1. Blue/Green
Two identical environments:

```
Blue (live)
Green (new version)
```

Instant cutover.

### 2. Canary Deployment

```
5% traffic → new version
50% traffic → new version
100% traffic
```

### 3. Shadow Deployment
Shadow requests to new version (no user impact).

---

# 15. INFRASTRUCTURE AS CODE

Use:

- Terraform  
- Pulumi  
- AWS CDK  

Rules:

- Everything must be reproducible  
- Never mutate infrastructure manually  
- Keep state secure  

---

# 16. PRODUCTION SAFETY

- Circuit breakers  
- Load shedding  
- Retry budgets  
- Dashboards for:
  - CPU/Mem  
  - queue length  
  - p99 latency  
  - HPA metrics  
- SLO-based alerts  
- Autoscaling rules  

---

# END OF CHAPTER F
