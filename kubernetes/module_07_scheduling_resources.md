# Module 07: Scheduling, Affinity & Resource Management

## Why this matters for your profile
You manage heterogeneous clusters with GPU nodes, high-memory build nodes, and device farm nodes. Proper scheduling and resource governance prevents noisy neighbors and ensures CI/CT workloads get predictable execution.

## Concept Clarity

### Scheduling Decision Flow
1. Filtering: eliminate nodes that don't meet requirements (taints, resources, affinity)
2. Scoring: rank remaining nodes by preference
3. Binding: assign pod to highest-scoring node

### Resource Requests vs Limits
| Field | Purpose | Scheduling | Runtime |
|-------|---------|-----------|---------|
| requests | Guaranteed minimum | Used by scheduler | QoS classification |
| limits | Maximum allowed | Not used | Enforced (OOMKilled / throttled) |

### QoS Classes
| Class | Condition | Eviction Priority |
|-------|-----------|-------------------|
| Guaranteed | requests == limits for all containers | Last to be evicted |
| Burstable | requests < limits | Middle |
| BestEffort | No requests/limits set | First to be evicted |

### LimitRange & ResourceQuota
| Resource | Scope | Purpose |
|----------|-------|---------|
| LimitRange | Namespace | Default/min/max per pod/container |
| ResourceQuota | Namespace | Total resource cap for namespace |

### Scheduling Constraints
| Mechanism | Purpose |
|-----------|---------|
| nodeSelector | Simple label matching |
| nodeAffinity | Expressive node selection (required/preferred) |
| podAffinity | Co-locate with other pods |
| podAntiAffinity | Spread away from other pods |
| Taints & Tolerations | Nodes repel pods unless tolerated |
| Topology Spread | Even distribution across zones/nodes |
| Priority & Preemption | Critical workloads evict lower-priority ones |

### Taints & Tolerations
- Taint on node: `key=value:effect` (NoSchedule, PreferNoSchedule, NoExecute)
- Toleration on pod: allows scheduling despite taint

## Command Mastery

```bash
# Resource requests and limits
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: build-agent
spec:
  containers:
  - name: builder
    image: alpine
    command: ['sleep', '3600']
    resources:
      requests:
        cpu: "2"
        memory: "4Gi"
      limits:
        cpu: "4"
        memory: "8Gi"
EOF

# Check QoS class
kubectl get pod build-agent -o jsonpath='{.status.qosClass}'

# LimitRange
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: ci-limits
  namespace: ci
spec:
  limits:
  - type: Container
    default:
      cpu: "1"
      memory: "1Gi"
    defaultRequest:
      cpu: "500m"
      memory: "512Mi"
    max:
      cpu: "8"
      memory: "16Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
  - type: Pod
    max:
      cpu: "16"
      memory: "32Gi"
EOF

# ResourceQuota
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ci-quota
  namespace: ci
spec:
  hard:
    requests.cpu: "32"
    requests.memory: "64Gi"
    limits.cpu: "64"
    limits.memory: "128Gi"
    pods: "50"
    persistentvolumeclaims: "20"
EOF

kubectl describe resourcequota ci-quota -n ci

# Taints and Tolerations
kubectl taint nodes worker-1 dedicated=build:NoSchedule
kubectl taint nodes worker-2 dedicated=gpu:NoSchedule

# Pod with toleration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  nodeSelector:
    accelerator: nvidia-a100
  containers:
  - name: train
    image: nvidia/cuda:12.0-runtime-ubuntu22.04
    resources:
      limits:
        nvidia.com/gpu: 1
EOF

# Node Affinity
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: affinity-demo
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-type
            operator: In
            values: ["build", "general"]
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: ["us-central1-a"]
  containers:
  - name: app
    image: nginx
EOF

# Pod Anti-Affinity (spread replicas)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-spread
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-spread
  template:
    metadata:
      labels:
        app: web-spread
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: web-spread
            topologyKey: kubernetes.io/hostname
      containers:
      - name: web
        image: nginx
EOF

# Topology Spread Constraints
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: spread-demo
  labels:
    app: spread
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: spread
  containers:
  - name: app
    image: nginx
EOF

# Priority Classes
cat <<EOF | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-ci
value: 1000000
globalDefault: false
description: "Critical CI pipeline workloads"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-low
value: 100
description: "Low priority batch jobs"
EOF

# Check scheduling issues
kubectl describe pod <pending-pod> | grep -A 10 Events
kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory
kubectl top nodes
kubectl top pods -n ci
```

## Practical Lab

### Exercises
1. Create a namespace with ResourceQuota (20 CPU, 40Gi RAM) and LimitRange — deploy until quota is exceeded
2. Taint a node for "build-only" workloads — verify only tolerated pods schedule there
3. Implement pod anti-affinity to spread Jenkins agents across nodes
4. Create two PriorityClasses — demonstrate preemption when resources are scarce
5. Use topology spread constraints to distribute pods across availability zones
6. Simulate OOMKilled by setting a 64Mi limit and allocating 128Mi in the container

### Pass Criteria
- You can explain the scheduling flow end-to-end
- You understand QoS classes and eviction order
- You can design resource governance for a multi-team platform
- You know when to use each scheduling primitive

## Mock Interview Questions

1. **A pod is stuck in Pending. Walk me through your debugging process.**
2. **Design resource governance for a shared Kubernetes cluster with 5 teams running CI/CD workloads.**
3. **Explain the difference between requests and limits. What happens when a container exceeds each?**
4. **How would you ensure critical pipeline pods are never evicted in favor of batch jobs?**
5. **When would you use pod anti-affinity vs topology spread constraints?**
6. **A team complains their builds are slow. You suspect CPU throttling. How do you investigate?**
7. **How do taints/tolerations work on Talos clusters where you can't SSH into nodes?**
