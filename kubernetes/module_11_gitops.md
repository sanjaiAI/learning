# Module 11: GitOps (Argo CD, Flux CD)

## Why this matters for your profile
You drive GitOps adoption for declarative, auditable infrastructure and application delivery. This is how you manage CI/CD platform deployments across AKS, GKE, Talos, and OKD clusters.

## Concept Clarity

### GitOps Principles
1. **Declarative:** Entire system described in Git
2. **Versioned:** Git as single source of truth (audit trail)
3. **Automated:** Approved changes auto-applied to cluster
4. **Self-healing:** Drift detection and reconciliation

### Push vs Pull Model
| Model | Description | Example |
|-------|-------------|---------|
| Push | CI pipeline applies manifests to cluster | Jenkins `kubectl apply` |
| Pull | Agent in cluster pulls from Git | Argo CD, Flux CD |

### Argo CD Architecture
| Component | Purpose |
|-----------|---------|
| Application Controller | Reconciles desired vs actual state |
| Repo Server | Clones Git repos, renders manifests |
| API Server | REST/gRPC API, UI, CLI |
| ApplicationSet | Generate Applications from templates |
| Notifications | Alert on sync status changes |

### Flux CD Architecture
| Component | Purpose |
|-----------|---------|
| Source Controller | Fetches from Git/Helm/OCI repos |
| Kustomize Controller | Reconciles Kustomize overlays |
| Helm Controller | Reconciles HelmReleases |
| Notification Controller | Alerts and webhooks |
| Image Automation | Auto-update image tags in Git |

### Argo CD vs Flux CD
| Feature | Argo CD | Flux CD |
|---------|---------|---------|
| UI | Rich built-in UI | No built-in UI (Weave GitOps optional) |
| Multi-tenancy | AppProject isolation | Namespace scoping |
| Rendering | Helm, Kustomize, Jsonnet, plain | Kustomize, Helm |
| ApplicationSet | Powerful generation | Flux lacks equivalent |
| Image automation | Argo Image Updater | Built-in |
| CRD-based | Application, AppProject | GitRepository, Kustomization, HelmRelease |

### Sync Strategies
| Strategy | Description |
|----------|-------------|
| Manual | Requires manual sync trigger |
| Auto-sync | Applies changes automatically |
| Self-heal | Reverts manual kubectl changes |
| Prune | Deletes resources removed from Git |

## Command Mastery

### Argo CD

```bash
# Install Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# CLI login
argocd login localhost:8080 --insecure
argocd account update-password

# Create Application
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ci-platform
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/k8s-manifests.git
    targetRevision: main
    path: environments/production/ci-platform
  destination:
    server: https://kubernetes.default.svc
    namespace: ci
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
EOF

# ApplicationSet (multi-cluster)
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: platform-services
  namespace: argocd
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          environment: production
  template:
    metadata:
      name: '{{name}}-monitoring'
    spec:
      project: platform
      source:
        repoURL: https://github.com/org/platform-charts.git
        targetRevision: main
        path: monitoring
      destination:
        server: '{{server}}'
        namespace: monitoring
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
EOF

# AppProject for isolation
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: ci-team
  namespace: argocd
spec:
  description: CI/CD team project
  sourceRepos:
  - 'https://github.com/org/ci-*'
  destinations:
  - namespace: 'ci-*'
    server: https://kubernetes.default.svc
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  namespaceResourceWhitelist:
  - group: '*'
    kind: '*'
EOF

# CLI operations
argocd app list
argocd app get ci-platform
argocd app sync ci-platform
argocd app diff ci-platform
argocd app history ci-platform
argocd app rollback ci-platform 2
```

### Flux CD

```bash
# Bootstrap Flux
flux bootstrap github \
  --owner=my-org \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/production \
  --personal

# GitRepository source
cat <<EOF | kubectl apply -f -
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: platform-repo
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/org/platform-manifests
  ref:
    branch: main
  secretRef:
    name: git-credentials
EOF

# Kustomization (reconcile path from Git)
cat <<EOF | kubectl apply -f -
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: ci-platform
  namespace: flux-system
spec:
  interval: 5m
  path: ./environments/production/ci
  prune: true
  sourceRef:
    kind: GitRepository
    name: platform-repo
  healthChecks:
  - apiVersion: apps/v1
    kind: Deployment
    name: jenkins
    namespace: ci
  timeout: 3m
EOF

# HelmRelease
cat <<EOF | kubectl apply -f -
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: prometheus
  namespace: monitoring
spec:
  interval: 10m
  chart:
    spec:
      chart: kube-prometheus-stack
      version: ">=45.0.0 <46.0.0"
      sourceRef:
        kind: HelmRepository
        name: prometheus-community
        namespace: flux-system
  values:
    prometheus:
      prometheusSpec:
        retention: 30d
EOF

# Flux CLI
flux get all
flux get kustomizations
flux reconcile kustomization ci-platform
flux logs --kind=Kustomization --name=ci-platform
flux suspend kustomization ci-platform
flux resume kustomization ci-platform
```

## Practical Lab

### Exercises
1. Install Argo CD — create an Application pointing to a Git repo with K8s manifests
2. Enable auto-sync with self-heal — make a manual `kubectl` change and watch it revert
3. Create an ApplicationSet that deploys to multiple namespaces from a single source
4. Install Flux — bootstrap and deploy a HelmRelease
5. Implement a promotion workflow: dev → staging → production via Git PRs
6. Set up Argo CD notifications to Slack on sync failures

### Pass Criteria
- You can set up GitOps from scratch with either Argo CD or Flux
- You understand sync strategies and their implications
- You can design multi-environment promotion workflows
- You know how to handle secrets in GitOps (Sealed Secrets, SOPS, External Secrets)

## Mock Interview Questions

1. **Explain GitOps principles. Why is the pull model better than push for production?**
2. **Compare Argo CD and Flux CD. When would you choose each?**
3. **How do you handle secrets in a GitOps workflow where everything is in Git?**
4. **Design a multi-cluster GitOps architecture for dev/staging/production environments.**
5. **What happens when someone makes a manual kubectl change in a GitOps-managed cluster?**
6. **How would you implement progressive delivery (canary/blue-green) with GitOps?**
7. **A sync is failing. How do you debug? What are common failure modes?**
