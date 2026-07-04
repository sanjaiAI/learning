# Module 05: Configuration & Secrets
# மாடுல் 05: Configuration & Secrets (அமைப்பு & ரகசியங்கள்)

---

## 🎯 What? | என்ன?

**English:** ConfigMaps store non-sensitive config. Secrets store sensitive data (passwords, tokens). Both inject configuration into pods without hardcoding.

**தமிழ்:** ConfigMaps = sensitive இல்லாத config (DB host, log level). Secrets = sensitive data (passwords, API keys). இரண்டும் hardcode இல்லாமல் pods-க்கு config inject செய்கின்றன.

### Analogy | உதாரணம்
> ConfigMap = Office notice board (everyone can read) | அறிவிப்பு பலகை
> Secret = Locker with key (restricted access) | பூட்டிய locker
> Both avoid writing directly on the wall (hardcoding) | நேரடியாக wall-ல் எழுதாதீர்கள்

---

## ⚠️ Important Truth | முக்கிய உண்மை

**Kubernetes Secrets are NOT encrypted by default!** They're only base64-encoded (anyone can decode).

**தமிழ்:** K8s Secrets default-ஆ encrypt ஆகவில்லை! base64 மட்டுமே — யாரும் decode செய்யலாம்.

```bash
# "Secrets" are easily decoded — NOT truly secret!
echo "cGFzc3dvcmQxMjM=" | base64 -d
# Output: password123    ← யாரும் படிக்கலாம்!
```

### Real Security = External Secret Stores

| Tool | How it integrates | உங்கள் அனுபவம் |
|------|------------------|----------------|
| **HashiCorp Vault** | CSI driver / Sidecar / ESO | ✅ You use this at EB |
| **Azure Key Vault** | CSI driver / Workload Identity | ✅ AKS clusters |
| **GCP Secret Manager** | CSI driver / Workload Identity | ✅ GKE clusters |
| **External Secrets Operator** | Syncs external → K8s Secret | Industry standard |
| **Sealed Secrets** | Encrypted, safe for Git | GitOps-friendly |

---

## 🛠️ Commands | Commands

```bash
# --- ConfigMap ---
# Literal values-லிருந்து create
kubectl create configmap app-config \
  --from-literal=DB_HOST=postgres.svc \
  --from-literal=LOG_LEVEL=info

# File-லிருந்து create
kubectl create configmap nginx-conf --from-file=nginx.conf

# --- Secrets ---
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=MyP@ssw0rd

# TLS secret
kubectl create secret tls my-tls --cert=tls.crt --key=tls.key

# Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.io \
  --docker-username=user --docker-password=pass

# --- Pod-ல் use செய்வது ---
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: nginx
    # Method 1: Environment variables ஆக inject
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
    # Method 2: Files ஆக mount
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
      readOnly: true
    - name: secret-vol
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: config-vol
    configMap: {name: app-config}
  - name: secret-vol
    secret:
      secretName: db-creds
      defaultMode: 0400    # Read-only permissions
EOF

# --- Verify ---
kubectl exec app -- env | grep DB       # Env var check
kubectl exec app -- cat /etc/secrets/password  # File check

# --- External Secrets Operator example ---
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

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────────┐
│         CONFIG & SECRETS CHEAT SHEET                 │
├──────────────────────────────────────────────────────┤
│ ConfigMap = non-sensitive config (notice board)       │
│ Secret   = sensitive data (locker) — NOT encrypted!  │
│                                                      │
│ INJECT METHODS:                                      │
│   1. env vars   → envFrom / valueFrom                │
│   2. files      → volumeMounts                       │
│                                                      │
│ env var  = pod restart வேண்டும் update-க்கு          │
│ file mount = auto-updates (kubelet sync ~60s)        │
│                                                      │
│ REAL SECURITY:                                       │
│   Vault + ESO → K8s Secret → Pod                     │
│   Workload Identity → No static credentials          │
│   Sealed Secrets → Safe for Git                      │
│                                                      │
│ ENCRYPT AT REST:                                     │
│   EncryptionConfiguration on API server              │
│   Or use cloud KMS (Azure/GCP)                       │
└──────────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A | நேர்முகத் தேர்வு

**Q: K8s Secrets are base64, not encrypted. How do you secure them?**
- Encrypt at rest (EncryptionConfiguration + KMS)
- Use External Secrets Operator + Vault/Cloud KMS
- RBAC: restrict who can read secrets
- Audit logging on secret access

**Q: Env var vs volume mount — difference?**
- Env var: pod restart வேண்டும் update-க்கு. Simple to use.
- Volume: auto-updates (~60s). Better for files (certs, configs).

**Q: Team accidentally committed secret to Git. Response?**
1. Immediately rotate the credential
2. Remove from Git history (`git filter-branch` or BFG)
3. Audit access logs
4. Implement Gitleaks/pre-commit hooks to prevent

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] ConfigMap vs Secret difference explain முடியும்
- [ ] Secrets are NOT encrypted — explain why & fix
- [ ] Vault/ESO integration design செய்ய முடியும்
- [ ] env vs mount trade-offs சொல்ல முடியும்
