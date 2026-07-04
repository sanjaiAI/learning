# Module 13: Multi-cluster, Federation & Service Mesh

## Why this matters for your profile
You manage workloads across GKE, AKS, Talos, and OKD clusters. Understanding multi-cluster patterns, service mesh, and federation is critical for designing resilient, distributed platforms.

## Concept Clarity

### Multi-Cluster Patterns
| Pattern | Description | Use Case |
|---------|-------------|----------|
| Replicated | Same workloads on multiple clusters | HA, geo-distribution |
| Partitioned | Different workloads per cluster | Team isolation, regulation |
| Hybrid | Mix of cloud + on-prem | Your setup (GKE + Talos + AKS) |
| Failover | Active-passive cluster pair | Disaster recovery |

### Multi-Cluster Management Tools
| Tool | Approach |
|------|----------|
| Argo CD (ApplicationSet) | GitOps-based multi-cluster |
| Flux CD (multi-cluster) | GitOps with cluster-as-code |
| Rancher | Multi-cluster management UI/API |
| Admiralty | Virtual kubelet for cross-cluster scheduling |
| Liqo | Kubernetes multi-cluster fabric |
| Skupper | Layer 7 multi-cluster service connectivity |

### Service Mesh
| Mesh | Key Feature |
|------|-------------|
| Istio | Feature-rich, Envoy-based, complex |
| Linkerd | Lightweight, Rust-based proxy |
| Cilium Service Mesh | eBPF-based, no sidecar |
| Consul Connect | HashiCorp ecosystem |

### Service Mesh Capabilities
| Feature | Description |
|---------|-------------|
| mTLS | Automatic mutual TLS between services |
| Traffic management | Canary, blue-green, traffic splitting |
| Observability | Distributed tracing, metrics, access logs |
| Retry/timeout/circuit breaking | Resilience patterns |
| Authorization policies | Service-to-service access control |

### Istio Architecture
```
Control Plane: istiod (Pilot + Citadel + Galley)
Data Plane: Envoy sidecar proxies (injected per pod)

Traffic flow: Pod A → Envoy(A) → mTLS → Envoy(B) → Pod B
```

## Command Mastery

### Multi-Cluster with Argo CD

```bash
# Register clusters in Argo CD
argocd cluster add gke-prod --name gke-production
argocd cluster add aks-staging --name aks-staging
argocd cluster add talos-onprem --name talos-onprem

argocd cluster list

# ApplicationSet for multi-cluster deployment
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: monitoring-fleet
  namespace: argocd
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          tier: production
  template:
    metadata:
      name: '{{name}}-monitoring'
    spec:
      project: platform
      source:
        repoURL: https://github.com/org/platform.git
        targetRevision: main
        path: monitoring/base
      destination:
        server: '{{server}}'
        namespace: monitoring
      syncPolicy:
        automated:
          prune: true
EOF
```

### Istio Service Mesh

```bash
# Install Istio
istioctl install --set profile=default -y
kubectl label namespace default istio-injection=enabled

# Verify sidecars
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].name}' | tr ' ' '\n' | sort | uniq

# Traffic management — canary
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp
        subset: v1
      weight: 90
    - destination:
        host: myapp
        subset: v2
      weight: 10
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF

# mTLS (PeerAuthentication)
cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
EOF

# Authorization Policy
cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-ci-only
  namespace: production
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        namespaces: ["ci"]
    - source:
        principals: ["cluster.local/ns/ci/sa/deploy-agent"]
  - to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/*"]
EOF

# Observability
istioctl dashboard kiali
istioctl dashboard jaeger
istioctl analyze  # Configuration validation
```

### Linkerd (Lightweight alternative)

```bash
# Install Linkerd
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -
linkerd check

# Inject sidecars
kubectl get deploy -n ci -o yaml | linkerd inject - | kubectl apply -f -

# Dashboard
linkerd viz install | kubectl apply -f -
linkerd viz dashboard

# Traffic split
cat <<EOF | kubectl apply -f -
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: myapp-split
spec:
  service: myapp
  backends:
  - service: myapp-v1
    weight: 900m
  - service: myapp-v2
    weight: 100m
EOF
```

## Practical Lab

### Exercises
1. Register multiple clusters (kind clusters) in Argo CD and deploy across them
2. Install Istio — verify mTLS between services using `istioctl proxy-config`
3. Implement canary deployment with traffic splitting (90/10)
4. Create an AuthorizationPolicy that restricts service-to-service communication
5. Install Linkerd and compare resource overhead with Istio
6. Design a multi-cluster architecture for your CI/CD platform (GKE + Talos + AKS)

### Pass Criteria
- You can articulate multi-cluster strategies and trade-offs
- You understand service mesh data plane vs control plane
- You can implement mTLS and traffic management
- You know when a service mesh adds value vs unnecessary complexity

## Mock Interview Questions

1. **Design a multi-cluster strategy for an organization running workloads across GKE, AKS, and on-prem Talos.**
2. **When would you introduce a service mesh? When is it overkill?**
3. **Compare Istio, Linkerd, and Cilium Service Mesh. Trade-offs?**
4. **How does mTLS work in a service mesh? How is certificate rotation handled?**
5. **Explain canary deployments with traffic splitting. How do you measure success?**
6. **How would you implement cross-cluster service discovery?**
7. **What are the performance implications of sidecar injection? How do you mitigate?**
