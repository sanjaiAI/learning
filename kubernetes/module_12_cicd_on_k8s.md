# Module 12: CI/CD Workloads on Kubernetes
# மாடுல் 12: Kubernetes-ல் CI/CD Workloads

---

## 🎯 What? | என்ன?

**English:** Running your CI/CD infrastructure (Jenkins, build agents, test runners) ON Kubernetes — dynamic scaling, isolation, and self-healing for pipelines.

**தமிழ்:** CI/CD infrastructure-ஐ (Jenkins, build agents, test runners) Kubernetes-ல் run செய்வது — dynamic scaling, isolation, self-healing.

### Analogy | உதாரணம்
> Old way: Fixed number of build machines (always running, costly)
> K8s way: Spawn build pods on demand, destroy when done (pay only when building)

> பழைய முறை: நிலையான build machines (எப்போதும் run, costly)
> K8s முறை: Build-க்கு தேவையானபோது pods create, முடிந்ததும் destroy (on-demand)

---

## 📊 Patterns | முறைகள்

| Pattern | How it works | Your experience |
|---------|-------------|-----------------|
| **Jenkins + K8s plugin** | Controller spawns agent pods on demand | ✅ You do this |
| **GitHub Actions ARC** | Self-hosted runners as K8s pods | ✅ Certified |
| **Tekton** | Cloud-native pipelines (CRD-based) | Awareness |
| **Kaniko/Buildah** | Build container images WITHOUT Docker daemon | ✅ Security best practice |

### Image Build: Why NOT Docker-in-Docker?

| Method | Security | தமிழ் |
|--------|----------|-------|
| Docker-in-Docker | ❌ Privileged (full host access) | Full host access = dangerous! |
| **Kaniko** ✓ | ✅ Unprivileged, userspace build | Secure — no special permissions needed |
| **Buildah** ✓ | ✅ Rootless OCI builds | Secure — root இல்லாமல் build |

---

## 🛠️ Commands | Commands

### Jenkins on K8s

```bash
# Install Jenkins
helm install jenkins jenkins/jenkins -n ci --create-namespace -f - <<EOF
controller:
  resources:
    requests: {cpu: "1", memory: "2Gi"}
agent:
  resources:
    requests: {cpu: "500m", memory: "1Gi"}
    limits: {cpu: "2", memory: "4Gi"}
persistence:
  enabled: true
  size: 50Gi
EOF

# Jenkinsfile with K8s agent (உங்கள் daily work)
# pipeline {
#   agent {
#     kubernetes {
#       yaml '''
#       apiVersion: v1
#       kind: Pod
#       spec:
#         containers:
#         - name: build
#           image: gradle:7-jdk17
#           resources:
#             requests: {cpu: "2", memory: "4Gi"}
#         - name: kaniko
#           image: gcr.io/kaniko-project/executor:debug
#       '''
#     }
#   }
# }
```

### Kaniko (Secure image build)

```bash
# Build image without Docker daemon
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: kaniko-build
  namespace: ci
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    args:
    - "--dockerfile=Dockerfile"
    - "--context=git://github.com/org/app.git"
    - "--destination=myregistry.io/app:v1.0"
    - "--cache=true"                    # Layer caching (faster!)
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker
  volumes:
  - name: docker-config
    secret:
      secretName: regcred
      items: [{key: .dockerconfigjson, path: config.json}]
  restartPolicy: Never
EOF
```

### GitHub Actions Runner Controller (ARC)

```bash
# Install ARC
helm install arc actions-runner-controller/actions-runner-controller \
  -n actions-runner-system --create-namespace

# Auto-scaling runners
cat <<EOF | kubectl apply -f -
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: runners
spec:
  replicas: 2
  template:
    spec:
      organization: my-org
      labels: [self-hosted, linux, k8s]
---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: runners-scaler
spec:
  scaleTargetRef: {name: runners}
  minReplicas: 1
  maxReplicas: 10
  scaleUpTriggers:
  - githubEvent: {workflowJob: {}}
    duration: "30m"
EOF
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│        CI/CD ON K8s CHEAT SHEET                  │
├──────────────────────────────────────────────────┤
│ JENKINS ON K8S:                                  │
│   Controller (StatefulSet) → Dynamic agents      │
│   Agents = ephemeral pods (spawn & destroy)      │
│                                                  │
│ IMAGE BUILDS (secure):                           │
│   ❌ Docker-in-Docker (privileged, risky)        │
│   ✓ Kaniko (unprivileged, userspace)             │
│   ✓ Buildah (rootless, OCI-compliant)            │
│                                                  │
│ GITHUB ACTIONS ON K8S:                           │
│   ARC (Runner Controller) + auto-scaling         │
│                                                  │
│ ISOLATION:                                       │
│   Namespace per team                             │
│   ResourceQuota per namespace                    │
│   NetworkPolicy (no cross-team access)           │
│   RBAC (team sees only own namespace)            │
│                                                  │
│ CACHING:                                         │
│   PVC for build cache (gradle, npm, maven)       │
│   Kaniko --cache=true (layer cache)              │
└──────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A | நேர்முகத் தேர்வு

**Q: Why Docker-in-Docker is bad for CI?**
- Privileged mode = container has full host access. Security risk. If compromised, attacker owns the node.
- Alternative: Kaniko (userspace, no privileges), Buildah (rootless).

**Q: How to handle build caching in ephemeral pods?**
- PVC mounted to agent pods for build cache (gradle ~/.gradle, npm node_modules)
- Kaniko `--cache=true` + `--cache-repo` for image layer caching
- Object storage (GCS/S3) for distributed cache

**Q: Design CI platform for 10 teams on K8s?**
- Namespace per team + RBAC + ResourceQuota + NetworkPolicy
- Shared Jenkins controller, team-specific agent templates
- Priority classes (release builds > feature builds)
- Auto-scaling agent pods based on queue depth

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] Jenkins on K8s architecture explain முடியும்
- [ ] Kaniko vs DinD security explain முடியும்
- [ ] ARC setup செய்ய முடியும்
- [ ] Multi-team CI isolation design முடியும்
