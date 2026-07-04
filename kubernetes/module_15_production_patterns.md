# Module 15: Production Patterns & Platform Engineering

## Why this matters for your profile
As a Technical Architect, you design the platform that other teams consume. This module covers production-grade patterns: multi-tenancy, platform APIs, developer experience, cost governance, and cluster lifecycle management — especially across GKE, AKS, Talos, and OKD.

## Concept Clarity

### Platform Engineering Principles
| Principle | Description |
|-----------|-------------|
| Self-service | Teams deploy without tickets (golden paths) |
| Guardrails, not gates | Policies prevent bad outcomes without blocking velocity |
| Abstraction | Hide infra complexity behind platform APIs |
| Observability-first | Built-in metrics, logs, traces for every workload |
| Cattle, not pets | Immutable, reproducible infrastructure |

### Multi-Tenancy Models
| Model | Isolation | Overhead |
|-------|-----------|----------|
| Namespace-based | Logical (shared cluster) | Low |
| Virtual cluster (vCluster) | API-level isolation | Medium |
| Dedicated cluster per tenant | Full isolation | High |

### Cluster Lifecycle Management
| Platform | Lifecycle Tool |
|----------|---------------|
| GKE | Terraform + GKE Autopilot/Standard |
| AKS | Terraform + Azure policies |
| Talos | talosctl + GitOps (machine configs in Git) |
| OKD | Installer + Machine API |
| Generic | Cluster API (CAPI) |

### Talos Linux — Immutable Kubernetes OS
| Feature | Description |
|---------|-------------|
| No SSH | API-only management (talosctl) |
| Immutable | Read-only root filesystem |
| Minimal | No package manager, no shell |
| Declarative | Machine configuration as YAML |
| Secure by default | No open ports except API |

### Cost Governance
| Strategy | Tool |
|----------|------|
| Resource right-sizing | VPA, Goldilocks, Kubecost |
| Spot/preemptible nodes | Node pools with spot toleration |
| Cluster autoscaler | Scale nodes based on pending pods |
| Namespace quotas | Prevent runaway resource consumption |
| Showback/chargeback | Kubecost, OpenCost per team |

### Upgrade Strategies
| Strategy | Description |
|----------|-------------|
| In-place | Upgrade existing nodes (rolling) |
| Blue-green cluster | New cluster → migrate workloads → decommission |
| Canary node pool | Upgrade subset of nodes first |

## Command Mastery

### Talos Linux Management

```bash
# Generate machine configs
talosctl gen config my-cluster https://control-plane:6443

# Apply config to node
talosctl apply-config --insecure --nodes <ip> --file controlplane.yaml

# Bootstrap cluster
talosctl bootstrap --nodes <control-plane-ip>

# Get kubeconfig
talosctl kubeconfig --nodes <control-plane-ip>

# Upgrade Talos OS
talosctl upgrade --nodes <ip> --image ghcr.io/siderolabs/installer:v1.7.0

# Upgrade Kubernetes
talosctl upgrade-k8s --nodes <control-plane-ip> --to 1.30.0

# Debug (no SSH!)
talosctl dmesg --nodes <ip>
talosctl logs kubelet --nodes <ip>
talosctl services --nodes <ip>
talosctl health --nodes <ip>
talosctl get members
talosctl dashboard --nodes <ip>

# etcd operations
talosctl etcd members --nodes <ip>
talosctl etcd snapshot /tmp/etcd.snapshot --nodes <ip>
```

### Cluster Autoscaler

```bash
# GKE autoscaler (via Terraform or gcloud)
# gcloud container clusters update my-cluster \
#   --enable-autoscaling --min-nodes=2 --max-nodes=20 --node-pool=build-pool

# AKS cluster autoscaler
# az aks update --resource-group rg --name aks-cluster \
#   --enable-cluster-autoscaler --min-count 2 --max-count 20

# Verify autoscaler
kubectl get pods -n kube-system | grep cluster-autoscaler
kubectl describe configmap cluster-autoscaler-status -n kube-system
kubectl get events | grep ScaledUp
```

### Vertical Pod Autoscaler (VPA) + Goldilocks

```bash
# Install VPA
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/latest/download/vertical-pod-autoscaler.yaml

# VPA recommendation
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: jenkins-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: jenkins
  updatePolicy:
    updateMode: "Off"  # Recommendation only
EOF

kubectl describe vpa jenkins-vpa

# Install Goldilocks for dashboard
helm install goldilocks fairwinds-stable/goldilocks -n goldilocks --create-namespace
kubectl label namespace ci goldilocks.fairwinds.com/enabled=true
```

### Multi-Tenancy with vCluster

```bash
# Install vCluster
helm repo add loft https://charts.loft.sh
helm install team-a-cluster loft/vcluster -n team-a --create-namespace

# Connect to virtual cluster
vcluster connect team-a-cluster -n team-a

# Each team gets full cluster experience but shares physical infra
kubectl get nodes  # Sees virtual nodes
kubectl create namespace my-app  # Fully isolated
```

### Platform API (Crossplane / Internal Developer Platform)

```bash
# Crossplane — provision infrastructure via K8s CRDs
helm install crossplane crossplane-stable/crossplane -n crossplane-system --create-namespace

# Composite Resource Definition (XRD) — platform API for teams
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xcienvs.platform.company.io
spec:
  group: platform.company.io
  names:
    kind: XCIEnvironment
    plural: xcienvs
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              teamName:
                type: string
              size:
                type: string
                enum: [small, medium, large]
EOF
```

### Production Hardening Checklist

```bash
# Pod Disruption Budgets
cat <<EOF | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: jenkins-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: jenkins
EOF

# Horizontal Pod Autoscaler
kubectl autoscale deployment web --cpu-percent=70 --min=3 --max=20

# Priority class for critical workloads
kubectl get priorityclasses

# Resource quotas per team
kubectl describe resourcequota -n team-a

# Network segmentation verification
kubectl get networkpolicy --all-namespaces
```

## Practical Lab

### Exercises
1. Set up a Talos cluster (3 nodes) using talosctl — bootstrap and verify
2. Implement namespace-based multi-tenancy with RBAC + ResourceQuota + NetworkPolicy + LimitRange
3. Configure cluster autoscaler and trigger scale-up with pending pods
4. Install VPA and Goldilocks — get right-sizing recommendations for existing workloads
5. Create a PodDisruptionBudget and test with `kubectl drain`
6. Design a platform API (Crossplane or custom CRD) that teams use to request CI environments

### Pass Criteria
- You can manage Talos clusters entirely via API (no SSH)
- You can design multi-tenant platforms with proper isolation
- You understand cluster lifecycle: provisioning → upgrading → decommissioning
- You can articulate platform engineering principles and golden paths
- You know cost governance strategies

## Mock Interview Questions

1. **Design a multi-tenant Kubernetes platform for 20 development teams. What isolation model would you choose and why?**
2. **How do you manage Talos clusters without SSH? What's your troubleshooting workflow?**
3. **Explain your cluster upgrade strategy. How do you minimize downtime?**
4. **How would you implement cost governance for a shared Kubernetes platform?**
5. **What's a PodDisruptionBudget and when is it critical?**
6. **Compare Cluster API (CAPI) vs Terraform for cluster lifecycle management.**
7. **Design a golden path for teams to onboard onto your platform. What do they get out of the box?**
8. **How do you right-size workloads? What tools and processes do you use?**
