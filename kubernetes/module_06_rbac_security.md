# Module 06: RBAC, Security Context & Pod Security

## Why this matters for your profile
You implement RBAC, workload identity, and admission control across AKS/GKE/Talos/OKD clusters. Interviewers will probe your ability to design least-privilege access for multi-tenant CI/CD platforms.

## Concept Clarity

### RBAC Components
| Resource | Scope | Purpose |
|----------|-------|---------|
| Role | Namespace | Grants permissions within a namespace |
| ClusterRole | Cluster-wide | Grants permissions across all namespaces |
| RoleBinding | Namespace | Binds Role/ClusterRole to subjects in a namespace |
| ClusterRoleBinding | Cluster-wide | Binds ClusterRole to subjects cluster-wide |

### Subjects
- User (external identity — no K8s User object)
- Group (e.g., `system:masters`, OIDC groups)
- ServiceAccount (in-cluster identity for pods)

### Verbs
`get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, `deletecollection`

### Security Context (Pod/Container level)
| Field | Purpose |
|-------|---------|
| runAsUser / runAsGroup | UID/GID for the process |
| runAsNonRoot | Reject containers running as root |
| readOnlyRootFilesystem | Immutable container filesystem |
| allowPrivilegeEscalation | Prevents `setuid` binaries |
| capabilities (add/drop) | Linux capabilities control |
| seccompProfile | Syscall filtering |

### Pod Security Standards (PSS) — replaces PodSecurityPolicy
| Level | Description |
|-------|-------------|
| Privileged | Unrestricted (system workloads) |
| Baseline | Minimally restrictive (prevent known escalations) |
| Restricted | Heavily restricted (best practice) |

### OKD/OpenShift Security Context Constraints (SCCs)
- Similar to PSS but more granular
- `restricted`, `anyuid`, `privileged`, `hostnetwork`
- Bound to ServiceAccounts or users

## Command Mastery

```bash
# Create ServiceAccount
kubectl create serviceaccount ci-agent -n ci

# Create Role
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-manager
  namespace: ci
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["get", "list", "create", "delete"]
EOF

# Bind Role to ServiceAccount
kubectl create rolebinding ci-agent-binding \
  --role=pod-manager \
  --serviceaccount=ci:ci-agent \
  -n ci

# ClusterRole for read-only cluster access
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-viewer
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps", "batch"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
EOF

# Test permissions
kubectl auth can-i create pods --as=system:serviceaccount:ci:ci-agent -n ci
kubectl auth can-i delete nodes --as=system:serviceaccount:ci:ci-agent
kubectl auth whoami  # K8s 1.27+

# Security Context
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: ci
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:1.25
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
EOF

# Pod Security Admission (namespace labels)
kubectl label namespace ci pod-security.kubernetes.io/enforce=restricted
kubectl label namespace ci pod-security.kubernetes.io/warn=restricted
kubectl label namespace ci pod-security.kubernetes.io/audit=restricted

# Verify — try deploying a privileged pod (should be rejected)
kubectl run priv --image=nginx --overrides='{"spec":{"containers":[{"name":"priv","image":"nginx","securityContext":{"privileged":true}}]}}' -n ci
```

## Practical Lab

### Exercises
1. Create a multi-tenant cluster with namespaces `team-a` and `team-b` — each team can only manage their own resources
2. Create a CI agent ServiceAccount that can create/delete pods and jobs but nothing else
3. Apply Pod Security Standards (`restricted`) to a namespace and try deploying non-compliant pods
4. Implement a security context that drops all capabilities and runs as non-root
5. Audit existing RBAC: `kubectl get clusterrolebindings -o wide | grep -v system:`
6. (OKD) Create a custom SCC and bind it to a ServiceAccount

### Pass Criteria
- You can design least-privilege RBAC from scratch
- You understand the difference between Role/ClusterRole and when each is appropriate
- You can harden a pod spec to pass `restricted` PSS
- You can explain how OKD SCCs differ from K8s PSS/PSA

## Mock Interview Questions

1. **Design an RBAC model for a multi-team CI/CD platform where each team should only access their own namespace.**
2. **What's the difference between a SecurityContext at pod level vs container level?**
3. **A developer says their pod won't start after you enforced restricted Pod Security. How do you debug and guide them?**
4. **How does workload identity work in GKE/AKS? Why is it better than static ServiceAccount keys?**
5. **Explain the principle of least privilege applied to Kubernetes. Give examples from your work.**
6. **What are SCCs in OpenShift and how do they compare to Pod Security Standards?**
7. **A ServiceAccount has cluster-admin. What's the risk and how would you remediate?**
