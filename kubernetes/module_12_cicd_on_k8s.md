# Module 12: CI/CD Workloads on Kubernetes

## Why this matters for your profile
You deploy Jenkins, build agents, test runners, and Android emulators as Kubernetes workloads. This module covers patterns for running CI/CD infrastructure on K8s — dynamic agents, resource isolation, and pipeline execution.

## Concept Clarity

### CI/CD Platform Patterns on Kubernetes
| Pattern | Description |
|---------|-------------|
| Controller + Dynamic Agents | Jenkins controller spawns pod-based agents on demand |
| Tekton / Argo Workflows | Cloud-native pipeline execution (CRD-based) |
| GitHub Actions Runner Controller (ARC) | Self-hosted runners on K8s |
| GitLab Runner on K8s | GitLab CI executor using K8s pods |
| Buildah/Kaniko | In-cluster container builds (no Docker socket) |

### Jenkins on Kubernetes
```
Jenkins Controller (StatefulSet)
    ↓ (Kubernetes plugin)
Dynamic Agent Pods (ephemeral)
    ├── Build container (gradle/maven/make)
    ├── Sidecar: docker/kaniko (for image builds)
    └── Sidecar: gcloud/az (for cloud operations)
```

### Container Image Build Strategies (No Docker-in-Docker)
| Tool | Approach | Privilege |
|------|----------|-----------|
| Kaniko | Userspace image build | Unprivileged |
| Buildah | OCI-compliant, rootless | Unprivileged |
| Docker-in-Docker | Docker daemon in container | Privileged (avoid) |
| BuildKit | Moby buildkit daemon | Rootless option |

### Tekton Pipeline Architecture
| CRD | Purpose |
|-----|---------|
| Task | Sequence of Steps (containers) |
| TaskRun | Execution of a Task |
| Pipeline | DAG of Tasks |
| PipelineRun | Execution of a Pipeline |
| Workspace | Shared storage between Tasks |
| Trigger | Event-driven pipeline execution |

### Resource Isolation for CI Workloads
| Concern | Solution |
|---------|----------|
| Noisy neighbor | ResourceQuota, LimitRange per team namespace |
| Security | Pod Security Standards, no privileged, RBAC |
| Build secrets | Vault injection, short-lived tokens |
| Artifact storage | PVC or object storage (GCS/Azure Blob) |
| Network isolation | NetworkPolicy per CI namespace |

## Command Mastery

### Jenkins on Kubernetes

```bash
# Install Jenkins via Helm
helm repo add jenkins https://charts.jenkins.io
helm install jenkins jenkins/jenkins -n ci --create-namespace -f - <<EOF
controller:
  resources:
    requests:
      cpu: "1"
      memory: "2Gi"
    limits:
      cpu: "2"
      memory: "4Gi"
  installPlugins:
    - kubernetes:latest
    - workflow-aggregator:latest
    - git:latest
    - configuration-as-code:latest
agent:
  enabled: true
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2"
      memory: "4Gi"
persistence:
  enabled: true
  size: 50Gi
EOF

# Jenkins pod template for builds
# (configured via JCasC or UI)
# Example pipeline using K8s agents:
# pipeline {
#   agent {
#     kubernetes {
#       yaml '''
#         apiVersion: v1
#         kind: Pod
#         spec:
#           containers:
#           - name: build
#             image: gradle:7-jdk17
#             resources:
#               requests: { cpu: "2", memory: "4Gi" }
#           - name: kaniko
#             image: gcr.io/kaniko-project/executor:debug
#             command: ['sleep', '9999']
#       '''
#     }
#   }
#   stages { ... }
# }
```

### Kaniko (Rootless Image Builds)

```bash
# Build pod with Kaniko
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
    - "--context=git://github.com/org/myapp.git#refs/heads/main"
    - "--destination=myregistry.io/myapp:latest"
    - "--cache=true"
    - "--cache-repo=myregistry.io/cache"
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker
  volumes:
  - name: docker-config
    secret:
      secretName: regcred
      items:
      - key: .dockerconfigjson
        path: config.json
  restartPolicy: Never
EOF
```

### GitHub Actions Runner Controller (ARC)

```bash
# Install ARC
helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller
helm install arc actions-runner-controller/actions-runner-controller \
  -n actions-runner-system --create-namespace \
  --set authSecret.github_token="ghp_xxx"

# Runner Deployment (auto-scaling)
cat <<EOF | kubectl apply -f -
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: org-runners
  namespace: ci
spec:
  replicas: 3
  template:
    spec:
      organization: my-org
      labels:
      - self-hosted
      - linux
      - k8s
      resources:
        requests:
          cpu: "2"
          memory: "4Gi"
        limits:
          cpu: "4"
          memory: "8Gi"
---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: org-runners-autoscaler
  namespace: ci
spec:
  scaleTargetRef:
    name: org-runners
  minReplicas: 1
  maxReplicas: 10
  scaleUpTriggers:
  - githubEvent:
      workflowJob: {}
    duration: "30m"
EOF
```

### Tekton Pipelines

```bash
# Install Tekton
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

# Simple Task
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-and-push
spec:
  params:
  - name: image
    type: string
  workspaces:
  - name: source
  steps:
  - name: clone
    image: alpine/git
    script: |
      git clone https://github.com/org/myapp.git $(workspaces.source.path)
  - name: build
    image: gcr.io/kaniko-project/executor:latest
    args:
    - --dockerfile=$(workspaces.source.path)/Dockerfile
    - --context=$(workspaces.source.path)
    - --destination=$(params.image)
EOF

# Run the Task
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: build-run-001
spec:
  taskRef:
    name: build-and-push
  params:
  - name: image
    value: myregistry.io/myapp:v1
  workspaces:
  - name: source
    emptyDir: {}
EOF

tkn taskrun logs build-run-001
```

## Practical Lab

### Exercises
1. Deploy Jenkins on Kubernetes with dynamic pod-based agents
2. Create a Kaniko build pod that builds an image and pushes to a registry
3. Install GitHub Actions Runner Controller and run a workflow on self-hosted K8s runners
4. Create a Tekton Pipeline with clone → build → test → push stages
5. Implement resource quotas and network policies for CI namespaces
6. Design a multi-tenant CI platform: team isolation, shared build caches, artifact storage

### Pass Criteria
- You can run CI/CD pipelines natively on Kubernetes
- You understand why Docker-in-Docker is a security risk and know alternatives
- You can design auto-scaling for build agents
- You can implement proper isolation between teams' CI workloads

## Mock Interview Questions

1. **Design a CI/CD platform on Kubernetes for 10 teams. How do you handle isolation, scaling, and security?**
2. **Why is Docker-in-Docker problematic? What are the alternatives for building container images in K8s?**
3. **Compare Jenkins on K8s vs Tekton vs GitHub Actions (self-hosted). Trade-offs?**
4. **How do you handle build caching in ephemeral Kubernetes-based CI agents?**
5. **Your Jenkins agents are being OOMKilled. How do you investigate and fix?**
6. **How would you implement auto-scaling for CI workloads based on queue depth?**
7. **Explain how you'd run Android emulator tests as Kubernetes workloads (from your experience).**
