# Module 08: Helm Charts & Package Management

## Why this matters for your profile
You build reusable deployment patterns across SoC programs. Helm is the standard package manager — you use it for deploying Jenkins, monitoring stacks, Gerrit, and custom CI/CT applications on AKS/GKE/Talos/OKD.

## Concept Clarity

### Helm Architecture
| Component | Purpose |
|-----------|---------|
| Chart | Package of K8s resource templates |
| Release | Instance of a chart deployed to a cluster |
| Repository | HTTP server hosting chart archives |
| Values | Configuration override for templates |
| Templates | Go-templated K8s manifests |

### Chart Structure
```
mychart/
├── Chart.yaml          # Metadata (name, version, dependencies)
├── values.yaml         # Default configuration
├── charts/             # Subcharts (dependencies)
├── templates/          # Go-templated K8s manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl    # Named template partials
│   ├── NOTES.txt       # Post-install message
│   └── tests/
│       └── test-connection.yaml
└── .helmignore
```

### Helm Lifecycle
```
helm install → create → hook(pre-install) → resources → hook(post-install)
helm upgrade → hook(pre-upgrade) → update resources → hook(post-upgrade)
helm rollback → revert to previous release revision
helm uninstall → hook(pre-delete) → delete resources → hook(post-delete)
```

### Key Features
- **Values override**: `--set`, `--values`, multi-file merge
- **Hooks**: pre/post install/upgrade/delete, test
- **Dependencies**: subchart management via `Chart.yaml`
- **Library charts**: shared template logic (no resources deployed)
- **OCI support**: store charts in container registries

## Command Mastery

```bash
# Repository management
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm search repo nginx
helm search hub prometheus  # Search Artifact Hub

# Install
helm install my-nginx bitnami/nginx -n web --create-namespace
helm install jenkins jenkins/jenkins -f custom-values.yaml -n ci

# Show chart info before installing
helm show values bitnami/nginx | head -50
helm show chart bitnami/nginx

# Upgrade
helm upgrade my-nginx bitnami/nginx --set replicaCount=3 -n web
helm upgrade jenkins jenkins/jenkins -f prod-values.yaml -n ci

# Rollback
helm history my-nginx -n web
helm rollback my-nginx 1 -n web

# Dry run and template rendering
helm install test bitnami/nginx --dry-run --debug
helm template my-release ./mychart -f values.yaml > rendered.yaml

# Uninstall
helm uninstall my-nginx -n web

# Create a new chart
helm create mychart
helm lint mychart
helm package mychart
helm push mychart-0.1.0.tgz oci://registry.example.com/charts

# Dependency management
helm dependency update ./mychart
helm dependency build ./mychart
```

### Writing Charts — Key Patterns
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

```yaml
# values.yaml
replicaCount: 2
image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
nodeSelector: {}
tolerations: []
affinity: {}
```

### Helm Testing
```bash
# Test hook
cat <<EOF > templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "mychart.fullname" . }}-test"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
  - name: wget
    image: busybox
    command: ['wget', '--spider', 'http://{{ include "mychart.fullname" . }}:80']
  restartPolicy: Never
EOF

helm test my-release -n ci
```

## Practical Lab

### Exercises
1. Create a Helm chart for a CI build agent (Deployment + ServiceAccount + RBAC + ConfigMap)
2. Use `helm template` to render locally, inspect the output, then install
3. Perform an upgrade with `--set`, check history, then rollback
4. Add a subchart dependency (e.g., Redis for caching) and manage with `helm dependency`
5. Implement a pre-upgrade hook that runs a database migration Job
6. Package and push a chart to an OCI registry (use local registry for testing)

### Pass Criteria
- You can create a production-quality Helm chart from scratch
- You understand Go template syntax and named templates
- You can manage release lifecycle (install, upgrade, rollback, uninstall)
- You know when to use Helm vs Kustomize vs raw manifests

## Mock Interview Questions

1. **When would you use Helm vs Kustomize? What are the trade-offs?**
2. **Explain the Helm release lifecycle and what hooks are available.**
3. **How do you handle secrets in Helm charts? What are the options?**
4. **A helm upgrade failed midway. What's the state of the release? How do you recover?**
5. **Design a Helm chart library for your organization's CI/CD platform components.**
6. **How do you version and distribute internal Helm charts?**
7. **What's the `checksum/config` annotation pattern and why is it useful?**
