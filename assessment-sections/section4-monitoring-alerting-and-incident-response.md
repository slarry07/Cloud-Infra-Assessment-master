### Section 4: Monitoring, Alerting & Incident Response

#### 4.1 Observability stack

- **Metrics**
  - **Prometheus** (managed or self-hosted) scraping:
    - Node exporter, kube-state-metrics, app metrics.
  - **CloudWatch** for AWS service metrics (ALB, RDS, ELB, NAT, etc.).
  - Dashboards in **Grafana** combining both.

- **Logs**
  - Fluent Bit/Fluentd → CloudWatch Logs or Loki.
  - Structured JSON logs (app + NGINX + system).

- **Traces**
  - OpenTelemetry SDK in services.
  - Back-end: AWS X-Ray or Tempo/Jaeger.

- **K8s-specific**
  - Kube-state-metrics dashboards: pod status, HPA, deployments, etc.
  - Container Insights or custom dashboards for EKS.

#### 4.2 Alerting philosophy

- **Principles**
  - Alert on **user-facing symptoms** (SLOs: error rate, latency, availability).
  - Alert on a small set of core **resource causes** (exhaustion, saturation).
  - Other signals as **dashboards** or **low-priority notifications**, not paging.

- **Examples**
  - **SLO-based**:
    - p95 latency > X ms for Y minutes.
    - Error rate > Z% of traffic (5xx) for Y minutes.
  - **Infra**:
    - Node CPU > 90% for 10 min, or disk near full.
    - RDS `CPUUtilization` or `DatabaseConnections` approaching limits.
  - **K8s**:
    - Too many `CrashLoopBackOff` pods in namespace.
    - Node `NotReady` count > 0 in prod.

#### 4.3 Incident response process (simulated outage)

Simulated scenario: API outage; 5xx spike; all tenants impacted.

- **Step 1 – Detection & ack**
  - PagerDuty/On-call receives alert: 5xx > 5% for 5 minutes on `prod`.
  - On-call acknowledges within minutes.

- **Step 2 – Triage**
  - Check status page and dashboards: confirm broad incident.
  - Identify primary impacted services and regions.
  - Quick health check:

```bash
kubectl get pods -n prod
kubectl get nodes
kubectl get events -A --sort-by=.lastTimestamp | tail -n 50
```

- **Step 3 – Containment**
  - If a recent deployment correlates → **roll back** immediately.
  - If one feature is causing load spike → disable via feature flag or route traffic around it.
  - If region-specific → route traffic away (failover to secondary region if ready).

- **Step 4 – Communication**
  - Update internal Slack channel (incident war room).
  - Update status page with impact & initial assessment.
  - Regular updates every 15–30 minutes.

- **Step 5 – Resolution**
  - Apply the fix (rollback, patch, capacity change).
  - Monitor metrics to confirm recovery.

- **Step 6 – Post-incident review**
  - Within 24–72 hours, run a blameless postmortem:
    - Exact timeline (T0 detection, T1 mitigation, T2 resolution).
    - Root cause (technical + process).
    - Which alerts worked/failed.
    - Concrete action items with owners and deadlines.
