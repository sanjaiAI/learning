# Module 10: OPA Gatekeeper, Kyverno & Policy-as-Code

## Why this matters for your profile
You're an expert at OPA Gatekeeper — enforcing admission control, workload compliance, and resource governance. This is a core differentiator in your DevSecOps architect profile.

## Concept Clarity

### Admission Control Chain
```
kubectl apply → API Server → Authentication → Authorization → Mutating Webhooks → Validating Webhooks → etcd
                                                                    ↑                      ↑
                                                              (Kyverno mutate)     (Gatekeeper/Kyverno validate)
```

### OPA Gatekeeper Architecture
| Component | Purpose |
|-----------|---------|
| ConstraintTemplate | Defines the policy logic (Rego) |
| Constraint | Instance of a template with parameters |
| Audit | Background check of existing resources |
| Sync (Config) | Replicate resources for cross-resource policies |

### Kyverno Architecture
| Feature | Description |
|---------|-------------|
| ClusterPolicy / Policy | YAML-native policy (no Rego needed) |
| Validate | Block non-compliant resources |
| Mutate | Auto-inject defaults (labels, sidecars) |
| Generate | Auto-create resources (NetworkPolicy, ResourceQuota) |
| Verify Images | Cosign signature verification |

### Gatekeeper vs Kyverno
| Feature | Gatekeeper | Kyverno |
|---------|-----------|---------|
| Language | Rego | YAML (JMESPath) |
| Learning curve | Steeper | Easier |
| Mutation | Limited (v3.7+) | First-class |
| Generation | No | Yes |
| Image verification | External | Built-in |
| Maturity | Older, CNCF Graduated | Newer, CNCF Incubating |

### Common Policies (DevSecOps)
| Policy | Purpose |
|--------|---------|
| No privileged containers | Security baseline |
| Required labels | Governance, cost tracking |
| Allowed registries | Supply chain security |
| Resource limits required | Prevent resource starvation |
| No latest tag | Reproducibility |
| Read-only root filesystem | Container hardening |
| No hostPath volumes | Isolation |
| Image signature verification | Supply chain trust |

## Command Mastery

### OPA Gatekeeper

```bash
# Install Gatekeeper
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper gatekeeper/gatekeeper -n gatekeeper-system --create-namespace

# ConstraintTemplate — require labels
cat <<EOF | kubectl apply -f -
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels
      violation[{"msg": msg}] {
        provided := {label | input.review.object.metadata.labels[label]}
        required := {label | label := input.parameters.labels[_]}
        missing := required - provided
        count(missing) > 0
        msg := sprintf("Missing required labels: %v", [missing])
      }
EOF

# Constraint — enforce on all pods
cat <<EOF | kubectl apply -f -
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: pods-must-have-owner
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaceSelector:
      matchExpressions:
      - key: gatekeeper-ignore
        operator: DoesNotExist
  parameters:
    labels:
    - "team"
    - "app"
EOF

# ConstraintTemplate — allowed registries
cat <<EOF | kubectl apply -f -
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedregistries
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRegistries
      validation:
        openAPIV3Schema:
          type: object
          properties:
            registries:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8sallowedregistries
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not startswith_any(container.image, input.parameters.registries)
        msg := sprintf("Image '%v' is not from an allowed registry. Allowed: %v", [container.image, input.parameters.registries])
      }
      startswith_any(image, registries) {
        registry := registries[_]
        startswith(image, registry)
      }
EOF

# Constraint instance
cat <<EOF | kubectl apply -f -
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRegistries
metadata:
  name: allowed-registries
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    - apiGroups: ["apps"]
      kinds: ["Deployment", "StatefulSet", "DaemonSet"]
  parameters:
    registries:
    - "gcr.io/my-project/"
    - "myregistry.azurecr.io/"
    - "docker.io/library/"
EOF

# Check audit results
kubectl get k8srequiredlabels pods-must-have-owner -o yaml | grep -A 20 violations

# Test violation
kubectl run test --image=unauthorized.io/nginx  # Should be rejected
```

### Kyverno

```bash
# Install Kyverno
helm repo add kyverno https://kyverno.github.io/kyverno
helm install kyverno kyverno/kyverno -n kyverno --create-namespace

# Validate — require resource limits
cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-limits
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "CPU and memory limits are required."
      pattern:
        spec:
          containers:
          - resources:
              limits:
                cpu: "?*"
                memory: "?*"
EOF

# Mutate — inject default labels
cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-labels
spec:
  rules:
  - name: add-team-label
    match:
      any:
      - resources:
          kinds:
          - Pod
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            managed-by: "platform-team"
            environment: "{{request.namespace}}"
EOF

# Generate — auto-create NetworkPolicy for new namespaces
cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: generate-default-netpol
spec:
  rules:
  - name: default-deny
    match:
      any:
      - resources:
          kinds:
          - Namespace
    generate:
      kind: NetworkPolicy
      apiVersion: networking.k8s.io/v1
      name: default-deny-ingress
      namespace: "{{request.object.metadata.name}}"
      data:
        spec:
          podSelector: {}
          policyTypes:
          - Ingress
EOF

# Verify image signatures (Cosign)
cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  rules:
  - name: verify-cosign
    match:
      any:
      - resources:
          kinds:
          - Pod
    verifyImages:
    - imageReferences:
      - "myregistry.io/*"
      attestors:
      - entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              <your-cosign-public-key>
              -----END PUBLIC KEY-----
EOF

# Check policy reports
kubectl get policyreport -A
kubectl get clusterpolicyreport
```

## Practical Lab

### Exercises
1. Install Gatekeeper and create a ConstraintTemplate that blocks `latest` image tags
2. Write a Rego policy that requires all pods to have resource limits
3. Install Kyverno and create a mutation policy that adds a sidecar to all pods in a namespace
4. Create a Kyverno generate policy that creates a ResourceQuota for every new namespace
5. Implement an allowed-registries policy and verify it blocks unauthorized images
6. Run an audit of existing resources against your policies — remediate violations

### Pass Criteria
- You can write Rego policies for Gatekeeper from scratch
- You can create Kyverno policies for validate/mutate/generate
- You understand the audit vs enforce modes and rollout strategy
- You can design a policy-as-code strategy for a multi-tenant platform

## Mock Interview Questions

1. **Design a policy-as-code strategy for a multi-tenant CI/CD Kubernetes platform. What policies would you enforce?**
2. **Compare OPA Gatekeeper and Kyverno. When would you choose each?**
3. **How do you roll out a new policy without breaking existing workloads?**
4. **Write a Rego policy that ensures all containers come from approved registries.**
5. **How does Gatekeeper audit work? How do you handle existing violations?**
6. **Explain how image signature verification with Cosign/Sigstore works in an admission controller.**
7. **How would you implement policy exceptions for system namespaces?**
