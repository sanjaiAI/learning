# Module 04: Storage (PV, PVC, StorageClass, CSI)

## Why this matters for your profile
CI/CD workloads need persistent storage — build caches, artifact staging, test results. You manage both cloud (Azure Disk, GCP PD) and on-prem storage. Understanding CSI drivers and dynamic provisioning is essential for platform reliability.

## Concept Clarity

### Storage Hierarchy
```
StorageClass → defines HOW storage is provisioned
    ↓
PersistentVolume (PV) → actual storage resource
    ↓
PersistentVolumeClaim (PVC) → user's request for storage
    ↓
Pod → mounts PVC as volume
```

### Volume Types
| Type | Use Case |
|------|----------|
| emptyDir | Temp scratch space (dies with pod) |
| hostPath | Node-local path (avoid in production) |
| PersistentVolumeClaim | Durable storage |
| configMap / secret | Inject config as files |
| projected | Combine multiple sources |
| CSI | Cloud/vendor storage drivers |

### Access Modes
| Mode | Abbr | Description |
|------|------|-------------|
| ReadWriteOnce | RWO | Single node read/write |
| ReadOnlyMany | ROX | Many nodes read-only |
| ReadWriteMany | RWX | Many nodes read/write |
| ReadWriteOncePod | RWOP | Single pod (K8s 1.27+) |

### Reclaim Policies
| Policy | Action when PVC deleted |
|--------|------------------------|
| Retain | PV preserved, manual cleanup |
| Delete | PV and backing storage deleted |
| Recycle | Deprecated — basic scrub |

### CSI (Container Storage Interface)
- Standard API for storage vendors
- Examples: Azure Disk CSI, GCP PD CSI, EBS CSI, NFS CSI
- Enables snapshots, cloning, expansion

## Command Mastery

```bash
# Storage classes available
kubectl get storageclass
kubectl describe storageclass standard

# Create a PVC (dynamic provisioning)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: build-cache
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
EOF

# Check PVC and bound PV
kubectl get pvc
kubectl get pv
kubectl describe pvc build-cache

# Pod using PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: build-agent
spec:
  containers:
  - name: builder
    image: alpine
    command: ['sh', '-c', 'echo "cached data" > /cache/data && sleep 3600']
    volumeMounts:
    - name: cache-vol
      mountPath: /cache
  volumes:
  - name: cache-vol
    persistentVolumeClaim:
      claimName: build-cache
EOF

# emptyDir (shared between containers in a pod)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: shared-vol
spec:
  containers:
  - name: writer
    image: busybox
    command: ['sh', '-c', 'echo hello > /shared/data && sleep 3600']
    volumeMounts:
    - name: shared
      mountPath: /shared
  - name: reader
    image: busybox
    command: ['sh', '-c', 'cat /shared/data && sleep 3600']
    volumeMounts:
    - name: shared
      mountPath: /shared
  volumes:
  - name: shared
    emptyDir: {}
EOF

# Volume expansion (if storageclass allows)
kubectl patch pvc build-cache -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Static PV creation
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-artifacts
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteMany
  nfs:
    server: nfs.internal
    path: /exports/artifacts
  persistentVolumeReclaimPolicy: Retain
EOF

# Snapshots (CSI)
cat <<EOF | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: cache-snapshot
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: build-cache
EOF
```

## Practical Lab

### Exercises
1. Create a StorageClass, PVC, and Pod — verify data persists across pod restarts
2. Create a pod with emptyDir and demonstrate data sharing between sidecar containers
3. Test what happens when a PVC is deleted with `Retain` vs `Delete` reclaim policy
4. Create an NFS-backed PV with ReadWriteMany and mount it from multiple pods
5. Expand a PVC online and verify the filesystem grows
6. Design a storage strategy for CI build caches (fast ephemeral) vs artifact storage (durable)

### Pass Criteria
- You can explain the PV lifecycle (Available → Bound → Released → Retained/Deleted)
- You understand dynamic vs static provisioning
- You can choose the right access mode for each workload type
- You know when to use emptyDir vs PVC vs hostPath

## Mock Interview Questions

1. **Explain the relationship between StorageClass, PV, and PVC.**
2. **A pod is stuck in Pending because of storage. How do you debug?**
3. **How would you design shared storage for CI build caches across multiple build pods?**
4. **What's the difference between RWO and RWX? When does this matter in CI/CD?**
5. **How do CSI drivers work? How would you evaluate one for production?**
6. **Explain volume snapshots and how you'd use them for build artifact backup.**
7. **What are the security implications of hostPath volumes?**
