# Module 05: Configuration & Secrets Management

## Why this matters for your profile
You integrate HashiCorp Vault, manage pipeline credentials, and enforce zero-hardcoded-secrets policies. This module covers native K8s config management and how it integrates with external secret stores you already use.

## Concept Clarity

### ConfigMaps vs Secrets
| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| Purpose | Non-sensitive config | Sensitive data |
| Encoding | Plain text | Base64 (not encryption!) |
| Size limit | 1 MiB | 1 MiB |
| Mounting | Env vars or volume | Env vars or volume |
| At-rest encryption | No (unless enabled) | Optional (EncryptionConfiguration) |

### Secret Types
| Type | Usage |
|------|-------|
| Opaque | Generic key-value |
| kubernetes.io/tls | TLS cert + key |
| kubernetes.io/dockerconfigjson | Image pull credentials |
| kubernetes.io/service-account-token | SA token (legacy) |

### External Secret Management (Enterprise Patterns)
| Tool | Integration |
|------|-------------|
| HashiCorp Vault | CSI driver, sidecar injector, External Secrets Operator |
| Azure Key Vault | CSI driver, workload identity |
| GCP Secret Manager | CSI driver, workload identity |
| External Secrets Operator (ESO) | Syncs external secrets → K8s Secrets |
| Sealed Secrets (Bitnami) | Encrypted secrets safe for Git |

### Immutable ConfigMaps/Secrets
- `immutable: true` — prevents accidental updates, improves performance (no watch)

## Command Mastery

```bash
# ConfigMap from literal
kubectl create configmap app-config \
  --from-literal=DB_HOST=postgres.svc \
  --from-literal=LOG_LEVEL=info

# ConfigMap from file
echo "server.port=8080" > app.properties
kubectl create configmap app-file-config --from-file=app.properties

# Secret from literal
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=s3cur3P@ss

# TLS Secret
kubectl create secret tls my-tls \
  --cert=tls.crt --key=tls.key

# Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.io \
  --docker-username=user \
  --docker-password=pass

# Using ConfigMap and Secret in a Pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
spec:
  containers:
  - name: app
    image: nginx:1.25
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_HOST
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: password
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
      readOnly: true
    - name: secret-vol
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: config-vol
    configMap:
      name: app-config
  - name: secret-vol
    secret:
      secretName: db-creds
      defaultMode: 0400
EOF

# Verify
kubectl exec config-demo -- env | grep DB
kubectl exec config-demo -- cat /etc/secrets/password
kubectl exec config-demo -- ls -la /etc/secrets/

# Immutable ConfigMap
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: immutable-config
data:
  key: value
immutable: true
EOF

# envFrom (inject all keys)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: envfrom-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'env && sleep 3600']
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: db-creds
  restartPolicy: Never
EOF
```

### External Secrets Operator (ESO)
```bash
# ExternalSecret syncing from Vault/Azure/GCP
cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: app-secrets
  data:
  - secretKey: db-password
    remoteRef:
      key: secret/data/myapp
      property: password
EOF
```

### Secret encryption at rest
```bash
# Check if encryption is enabled
kubectl get apiserver -o yaml | grep -i encrypt

# EncryptionConfiguration example (for kubeadm)
# /etc/kubernetes/enc/enc.yaml
# apiVersion: apiserver.config.k8s.io/v1
# kind: EncryptionConfiguration
# resources:
# - resources: [secrets]
#   providers:
#   - aescbc:
#       keys:
#       - name: key1
#         secret: <base64-encoded-32-byte-key>
#   - identity: {}
```

## Practical Lab

### Exercises
1. Create a ConfigMap and Secret, mount both in a pod, and verify access
2. Update a ConfigMap mounted as a volume — observe how long the pod takes to see the change (kubelet sync period)
3. Create an immutable ConfigMap and try to update it — observe the error
4. Set up External Secrets Operator and sync a secret from a mock SecretStore
5. Create a docker-registry secret and use it as `imagePullSecrets`
6. Demonstrate why base64 is NOT encryption — decode a secret with `kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d`

### Pass Criteria
- You can explain why K8s Secrets are not truly secure by default
- You know the difference between env var injection vs volume mount (update behavior)
- You can design a secrets management strategy using Vault + ESO/CSI
- You understand encryption at rest configuration

## Mock Interview Questions

1. **Kubernetes Secrets are base64-encoded, not encrypted. How do you secure them in production?**
2. **Compare the approaches: Vault sidecar injector vs CSI driver vs External Secrets Operator. Which would you choose and why?**
3. **A team accidentally committed a Secret to Git. What's your incident response?**
4. **How do ConfigMap updates propagate to running pods? What's the delay?**
5. **Design a secrets management architecture for a multi-tenant CI/CD platform.**
6. **What's the difference between mounting a secret as a volume vs injecting as an env var? When would you prefer each?**
7. **How does workload identity eliminate the need for static credentials in cloud environments?**
