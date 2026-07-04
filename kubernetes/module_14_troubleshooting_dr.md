# Module 14: Troubleshooting & Disaster Recovery

## Why this matters for your profile
As a Technical Architect, you're the escalation point when things break. You need systematic debugging skills and DR strategies for production Kubernetes clusters — whether GKE, AKS, Talos, or OKD.

## Concept Clarity

### Troubleshooting Framework
```
1. Identify symptoms (what's broken?)
2. Gather data (events, logs, metrics, describe)
3. Isolate scope (cluster/node/namespace/pod/container)
4. Form hypothesis
5. Test and validate
6. Fix and document
```

### Common Failure Modes
| Symptom | Likely Cause |
|---------|-------------|
| Pod Pending | Insufficient resources, taints, affinity mismatch |
| CrashLoopBackOff | App crash, bad config, missing deps |
| ImagePullBackOff | Bad image name, auth failure, registry down |
| OOMKilled | Memory limit too low |
| Evicted | Node under pressure (disk/memory) |
| Node NotReady | Kubelet crash, network partition, disk full |
| Service unreachable | Endpoints empty, NetworkPolicy, DNS issue |
| Ingress 503 | No healthy backends, misconfigured path |

### Disaster Recovery Components
| Component | Backup Strategy |
|-----------|----------------|
| etcd | Snapshot backup (critical) |
| PersistentVolumes | CSI snapshots, Velero |
| Cluster state | GitOps (everything in Git) |
| Secrets | External secrets store (Vault) |
| Certificates | Cert-manager, manual backup of CA |

### Velero (Backup & Restore)
- Backs up K8s resources + PV data
- Supports scheduled backups
- Cross-cluster migration
- Selective restore (namespace, label)

## Command Mastery

### Systematic Debugging

```bash
# Cluster-level health
kubectl get nodes -o wide
kubectl top nodes
kubectl get componentstatuses  # deprecated but informative
kubectl get events --all-namespaces --sort-by=.metadata.creationTimestamp | tail -30

# Node issues
kubectl describe node <node-name> | grep -A 5 Conditions
kubectl describe node <node-name> | grep -A 10 "Allocated resources"
# Talos: no SSH — use talosctl
# talosctl dmesg --nodes <ip>
# talosctl services --nodes <ip>
# talosctl logs kubelet --nodes <ip>

# Pod debugging
kubectl get pods -n <ns> -o wide
kubectl describe pod <name> -n <ns>
kubectl logs <pod> -n <ns> --previous  # Previous crashed container
kubectl logs <pod> -c <container> -n <ns>
kubectl get events --field-selector involvedObject.name=<pod> -n <ns>

# Why is a pod Pending?
kubectl describe pod <name> | grep -A 5 "Events"
# Look for: FailedScheduling, Insufficient cpu/memory, node selector mismatch

# CrashLoopBackOff debugging
kubectl logs <pod> --previous
kubectl describe pod <pod> | grep -A 5 "Last State"
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'

# Exec into running container
kubectl exec -it <pod> -c <container> -- /bin/sh

# Debug with ephemeral container (K8s 1.25+)
kubectl debug <pod> -it --image=busybox --target=<container>

# Debug node (create privileged pod on node)
kubectl debug node/<node-name> -it --image=ubuntu

# Network debugging
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- /bin/bash
# Inside: curl, dig, nslookup, tcpdump, iperf, nmap

# DNS debugging
kubectl exec dnsutils -- nslookup kubernetes.default
kubectl exec dnsutils -- cat /etc/resolv.conf

# Service/Endpoint verification
kubectl get endpoints <service>
kubectl describe svc <service>

# Certificate issues
kubectl get csr
openssl s_client -connect <service-ip>:443 -servername <hostname>

# Resource pressure
kubectl top pods --sort-by=memory -n <ns>
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.qosClass}{"\n"}{end}'
```

### etcd Backup & Restore

```bash
# Backup (kubeadm clusters)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-$(date +%Y%m%d).db --write-table

# Restore
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-20240101.db \
  --data-dir=/var/lib/etcd-restored \
  --initial-cluster=master=https://master:2380

# Talos etcd backup
# talosctl etcd snapshot /backup/etcd.snapshot --nodes <control-plane-ip>
```

### Velero (Cluster Backup)

```bash
# Install Velero
velero install \
  --provider gcp \
  --bucket my-backup-bucket \
  --secret-file ./credentials-velero

# Backup namespace
velero backup create ci-backup --include-namespaces ci
velero backup describe ci-backup
velero backup logs ci-backup

# Scheduled backup
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces ci,monitoring \
  --ttl 720h

# Restore
velero restore create --from-backup ci-backup
velero restore create --from-backup ci-backup --include-resources pods,deployments

# Cross-cluster migration
# 1. Backup on source cluster
# 2. Configure Velero on target with same storage
# 3. Restore on target cluster
```

### Common Recovery Procedures

```bash
# Node NotReady — cordon and drain
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# Fix node, then:
kubectl uncordon <node>

# Force delete stuck pod
kubectl delete pod <name> --grace-period=0 --force

# Stuck namespace (finalizer issue)
kubectl get namespace <ns> -o json | jq '.spec.finalizers = []' | kubectl replace --raw "/api/v1/namespaces/<ns>/finalize" -f -

# Certificate renewal (kubeadm)
kubeadm certs check-expiration
kubeadm certs renew all
# Restart control plane components

# Recover from bad Deployment rollout
kubectl rollout undo deployment/<name>
kubectl rollout status deployment/<name>
```

## Practical Lab

### Exercises
1. Intentionally break a pod (bad image, OOM, missing configmap) and debug systematically
2. Create a node pressure scenario (fill disk) and observe eviction behavior
3. Backup etcd, delete a deployment, then restore from backup
4. Install Velero and perform a namespace backup + cross-namespace restore
5. Simulate network partition — kill CoreDNS and debug the symptoms
6. Practice using `kubectl debug` with ephemeral containers

### Pass Criteria
- You can systematically debug any pod/node/cluster issue
- You can perform etcd backup and restore
- You understand the DR strategy: GitOps + etcd backup + Velero + external secrets
- You can handle production incidents under pressure

## Mock Interview Questions

1. **A production pod is in CrashLoopBackOff at 3 AM. Walk me through your incident response.**
2. **Your etcd cluster lost quorum. What's the recovery procedure?**
3. **Design a disaster recovery strategy for a Kubernetes-based CI/CD platform.**
4. **A node is showing DiskPressure. What happens to pods? How do you fix it?**
5. **How do you debug networking issues when pods can't communicate?**
6. **What's your backup strategy for Kubernetes? What do you back up and how often?**
7. **On Talos (no SSH), how do you troubleshoot a node that's NotReady?**
