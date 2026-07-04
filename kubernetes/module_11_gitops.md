# Module 11: GitOps (Argo CD & Flux CD)
# மாடுல் 11: GitOps

---

## 🎯 What? | என்ன?

**English:** GitOps = Git is the single source of truth. Whatever is in Git, that's what runs in your cluster. Change Git → cluster auto-updates.

**தமிழ்:** GitOps = Git-ல் என்ன இருக்கிறதோ, அதுவே cluster-ல் run ஆகும். Git-ல் change செய்தால், cluster தானாக update ஆகும்.

### Analogy | உதாரணம்
> Google Docs: Whatever you type appears on everyone's screen automatically. Git = the document. Cluster = everyone's screen. Argo CD/Flux = auto-sync engine.

> Google Docs: நீங்கள் type செய்ததும் எல்லோர் screen-லும் appear ஆகிறது. Git = document. Cluster = screen. Argo CD = sync engine.

---

## 📊 Push vs Pull Model

| Model | How | Security | தமிழ் |
|-------|-----|----------|-------|
| **Push** (traditional) | CI pipeline runs `kubectl apply` | CI needs cluster credentials 🔓 | CI-க்கு cluster access வேண்டும் (risky) |
| **Pull** (GitOps) ✓ | Agent IN cluster pulls from Git | No external access needed ✅ | Cluster உள்ளே agent Git-லிருந்து pull செய்கிறது (safe) |

```mermaid
graph LR
    DEV[Developer] -->|git push| GIT[Git Repo]
    GIT -->|watches| ARGO[Argo CD / Flux<br/>Cluster-ல் உள்ளது]
    ARGO -->|applies| CLUSTER[Cluster Resources]
    ARGO -->|detects drift| GIT
```

---

## ⚔️ Argo CD vs Flux CD

| Feature | Argo CD | Flux CD |
|---------|---------|---------|
| UI | Beautiful built-in UI ✓ | No UI (CLI/Weave GitOps) |
| Learning curve | Moderate | Easy |
| Multi-cluster | ApplicationSet (powerful) | Multi-repo support |
| CRDs | Application, AppProject | GitRepository, Kustomization, HelmRelease |
| Best for | Visual teams, complex setups | CLI-first teams, simple setups |

---

## 🛠️ Commands | Commands

### Argo CD

```bash
# Install
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Create Application (Git-ல் இருக்கும் manifests deploy செய்)
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/k8s-manifests.git
    targetRevision: main
    path: apps/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true       # Git-லிருந்து remove ஆனால் cluster-லும் delete
      selfHeal: true    # Manual kubectl change revert ஆகும்!
EOF

# CLI operations
argocd app list
argocd app sync my-app       # Force sync
argocd app diff my-app       # What's different?
argocd app rollback my-app 2 # Previous version-க்கு போ
```

### Flux CD

```bash
# Bootstrap (Git repo-வுடன் connect)
flux bootstrap github \
  --owner=my-org --repository=fleet-infra \
  --branch=main --path=clusters/production

# Git source define
cat <<EOF | kubectl apply -f -
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-repo
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/org/manifests
  ref: {branch: main}
EOF

# Kustomization (reconcile from Git)
cat <<EOF | kubectl apply -f -
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 5m
  path: ./apps/production
  prune: true
  sourceRef: {kind: GitRepository, name: my-repo}
EOF

# CLI
flux get all
flux reconcile kustomization my-app  # Force sync
flux logs --kind=Kustomization --name=my-app
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│            GITOPS CHEAT SHEET                    │
├──────────────────────────────────────────────────┤
│ PRINCIPLES:                                      │
│   1. Git = single source of truth                │
│   2. Declarative (YAML in Git)                   │
│   3. Auto-applied (PR merge = deploy)            │
│   4. Self-healing (drift auto-fixed)             │
│                                                  │
│ ARGO CD SYNC MODES:                              │
│   Manual    = click to deploy                    │
│   Auto-sync = merge PR → auto deploy             │
│   Self-heal = kubectl change → reverted!         │
│   Prune     = delete from Git → delete cluster   │
│                                                  │
│ SECRETS IN GITOPS:                               │
│   ❌ Plain secrets in Git                        │
│   ✓ Sealed Secrets (encrypted in Git)            │
│   ✓ External Secrets Operator                    │
│   ✓ SOPS encryption                             │
│                                                  │
│ PROMOTION WORKFLOW:                              │
│   dev branch → PR → staging → PR → production    │
│   Git PR = deployment approval!                  │
└──────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A | நேர்முகத் தேர்வு

**Q: Why Pull model better than Push?**
- Push = CI needs cluster credentials (attack vector). Push doesn't detect drift.
- Pull = agent inside cluster (no external access needed). Auto-detects drift. Auditable via Git history.

**Q: Someone runs `kubectl edit` on a GitOps-managed resource?**
- Self-heal enabled → Argo/Flux reverts the manual change within sync interval.
- Git is truth. Manual changes overwritten.

**Q: How to handle secrets in GitOps?**
- Sealed Secrets (Bitnami): encrypt in Git, controller decrypts in cluster
- External Secrets Operator: reference external store (Vault), sync to K8s Secret
- SOPS + age/KMS: encrypt values.yaml files in Git

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] GitOps principles explain முடியும்
- [ ] Push vs Pull model compare முடியும்
- [ ] Argo CD Application create முடியும்
- [ ] Flux Kustomization create முடியும்
- [ ] Secrets strategy in GitOps explain முடியும்
