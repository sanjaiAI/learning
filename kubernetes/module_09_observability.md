# Module 09: Observability (Logging, Monitoring, Tracing)

## Why this matters for your profile
You design observability stacks (Prometheus, Grafana, ELK, AlertManager, New Relic) for CI/CT metrics, pipeline reliability tracking, and release stability. Interviewers will expect you to explain the three pillars and make architectural decisions.

## Concept Clarity

### Three Pillars of Observability
| Pillar | What | Tools |
|--------|------|-------|
| Metrics | Numeric time-series data | Prometheus, Thanos, VictoriaMetrics |
| Logs | Discrete event records | ELK, Loki, Fluentd/Fluent Bit |
| Traces | Request flow across services | Jaeger, Tempo, Zipkin, OpenTelemetry |

### Prometheus Architecture
```
Targets (pods/services) → Prometheus (scrape) → TSDB → PromQL → Grafana/AlertManager
                                                  ↓
                                             Thanos/Remote Write (long-term)
```

### Key Metrics Types
| Type | Description | Example |
|------|-------------|---------|
| Counter | Monotonically increasing | Total HTTP requests |
| Gauge | Can go up or down | Current memory usage |
| Histogram | Distribution of values | Request latency buckets |
| Summary | Similar to histogram, client-calculated | P50/P90/P99 latency |

### Kubernetes-Native Observability
| Component | Purpose |
|-----------|---------|
| Metrics Server | Basic resource metrics (kubectl top) |
| kube-state-metrics | Object state metrics (deployment replicas, pod status) |
| node-exporter | Hardware/OS metrics per node |
| cAdvisor | Container resource usage (built into kubelet) |

### Logging Architecture Patterns
| Pattern | Description |
|---------|-------------|
| Sidecar | Log container alongside app (per-pod) |
| DaemonSet | Node-level log collector (Fluent Bit) |
| Direct | App pushes logs to centralized store |

### SLI / SLO / SLA
| Term | Meaning | Example |
|------|---------|---------|
| SLI | Service Level Indicator | p99 latency, error rate |
| SLO | Service Level Objective | p99 < 500ms, 99.9% availability |
| SLA | Service Level Agreement | Contractual commitment + consequences |

## Command Mastery

```bash
# Install Prometheus stack via Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  -f - <<EOF
prometheus:
  prometheusSpec:
    retention: 15d
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
grafana:
  adminPassword: admin123
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
      - name: 'default'
        folder: ''
        type: file
        options:
          path: /var/lib/grafana/dashboards
alertmanager:
  config:
    route:
      receiver: 'slack'
    receivers:
    - name: 'slack'
      slack_configs:
      - channel: '#alerts'
        api_url: 'https://hooks.slack.com/services/xxx'
EOF

# Verify installation
kubectl get pods -n monitoring
kubectl get servicemonitor -n monitoring
kubectl get prometheusrule -n monitoring

# Port-forward to access
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n monitoring

# Custom ServiceMonitor for your app
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: ci-pipeline-metrics
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: jenkins
  endpoints:
  - port: metrics
    interval: 30s
    path: /prometheus
  namespaceSelector:
    matchNames:
    - ci
EOF

# PrometheusRule (alerting)
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ci-alerts
  namespace: monitoring
spec:
  groups:
  - name: ci-pipeline
    rules:
    - alert: PipelineFailureRateHigh
      expr: rate(ci_pipeline_failures_total[5m]) > 0.1
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Pipeline failure rate exceeds 10%"
    - alert: BuildQueueBacklog
      expr: ci_build_queue_length > 20
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Build queue backlog exceeds 20 jobs"
EOF

# PromQL queries
# Pod CPU usage
# sum(rate(container_cpu_usage_seconds_total{namespace="ci"}[5m])) by (pod)

# Memory usage percentage
# container_memory_working_set_bytes / container_spec_memory_limit_bytes * 100

# Pod restart rate
# rate(kube_pod_container_status_restarts_total[1h]) > 0

# Logging with Fluent Bit (DaemonSet)
helm repo add fluent https://fluent.github.io/helm-charts
helm install fluent-bit fluent/fluent-bit -n logging --create-namespace

# Loki for log aggregation
helm install loki grafana/loki-stack -n logging --set grafana.enabled=false

# Check logs
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit --tail=20

# Kubernetes events as metrics
kubectl get events --sort-by=.metadata.creationTimestamp -n ci
```

## Practical Lab

### Exercises
1. Deploy kube-prometheus-stack and access Grafana — explore default dashboards
2. Create a ServiceMonitor for a custom application exposing `/metrics`
3. Write PromQL queries: CPU usage by namespace, pod restart rate, memory pressure
4. Create a PrometheusRule with alerts for CI pipeline health
5. Deploy Fluent Bit as DaemonSet and verify logs reach your chosen backend
6. Define SLIs/SLOs for your CI/CD pipeline (build success rate, queue time, p95 duration)

### Pass Criteria
- You can deploy a complete observability stack from scratch
- You can write PromQL queries for production debugging
- You understand the ServiceMonitor/PrometheusRule CRD model
- You can design alerting that avoids alert fatigue

## Mock Interview Questions

1. **Design an observability strategy for a CI/CD platform running on Kubernetes.**
2. **Explain how Prometheus service discovery works in Kubernetes.**
3. **What's the difference between a histogram and a summary? When would you use each?**
4. **How would you define SLOs for a build pipeline? What SLIs would you track?**
5. **Your Prometheus is running out of memory. How do you investigate and fix?**
6. **Compare ELK vs Loki for log aggregation in Kubernetes. Trade-offs?**
7. **How do you prevent alert fatigue while ensuring critical issues are caught?**
