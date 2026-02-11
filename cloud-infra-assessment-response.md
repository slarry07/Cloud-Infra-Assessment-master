### Section 1: Cloud Architecture & Design (AWS EKS Multitenant SaaS)

#### 1.1 High-level architecture

**Goals**: public-facing, secure multitenancy, autoscaling, strong isolation, observable, DR-ready.

Conceptual diagram (simplified):

```mermaid
flowchart LR
  U[Users / Tenants] --> CF[CloudFront + WAF]
  CF --> ALB[Public ALB]
  ALB --> INGRESS[Ingress Controller (EKS)]
  
  subgraph VPC[App VPC]
    IGW[Internet Gateway]
    NAT[NAT Gateways (per AZ)]
    IGW <---> PublicSubnets
    NAT <---> PrivateSubnets

    subgraph PublicSubnets[Public Subnets]
      ALB
    end

    subgraph PrivateSubnets[Private Subnets - EKS Node Groups]
      EKS[EKS Cluster]
      EKS --> SVC[Services / Pods]
    end

    subgraph DataSubnets[Data Subnets]
      RDS[(RDS/Aurora)]
      Redis[(ElastiCache)]
      S3Buckets[(S3 w/ KMS)]
    end
  end

  EKS --> RDS
  EKS --> Redis
  EKS --> S3Buckets

  subgraph Observability
    CW[CloudWatch]
    PM[Prometheus]
    GF[Grafana]
    LG[Loki / CloudWatch Logs]
  end

  EKS --> PM
  EKS --> LG
  AWSBackup[(AWS Backup)] --> RDS
  AWSBackup --> S3Buckets
```

#### 1.2 Networking

- **VPC & subnets**
  - **Design**: 1 VPC per environment (`dev`, `stg`, `prod`), 3+ AZs.
  - **Subnets**:
    - **Public**: ALB, NAT Gateways, bastion (if needed).
    - **Private (app)**: EKS node groups (no public IPs).
    - **Private (data)**: RDS/Aurora, ElastiCache, internal services.
  - **Routing**:
    - Public subnets → IGW; private subnets → NAT for outbound internet.
    - Security groups & NACLs to restrict L3/L4 access.

- **Ingress / egress**
  - **Ingress**: CloudFront + AWS WAF → ALB (or NLB if gRPC), ALB → NGINX or AWS Load Balancer Controller Ingress in EKS.
  - **Service exposure**:
    - Public APIs via `Ingress` + `LoadBalancer` Services.
    - Internal services via `ClusterIP` + `Service` mesh (e.g., AWS App Mesh or Istio if needed).

- **Multitenancy routing**
  - **Options**:
    - **Host-based routing** (`tenantA.example.com`) + tenant ID in JWT.
    - **Path-based routing** (`/tenant/<id>/...`) at app level.
  - Network isolation for “premium” tenants: dedicated namespaces, node pools, or even dedicated clusters.

#### 1.3 IAM & security model

- **Identity**
  - **Humans**: SSO (AWS SSO / IAM Identity Center) + least-privilege IAM roles; no long-lived IAM users.
  - **Workloads**: IAM Roles for Service Accounts (IRSA) per app, not node IAM.

- **Access patterns**
  - EKS cluster role for CI/CD deployer, read-only role for SRE/Support, break-glass admin role with MFA + strong approval.
  - Per-service IAM policies:
    - App service role can only access its own S3 prefix (`arn:aws:s3:::app-data/tenant/*`), specific KMS keys, specific RDS cluster, etc.

- **Secrets**
  - Secrets in **AWS Secrets Manager** or **SSM Parameter Store**.
  - Mounted via CSI driver or injected via env vars; KMS-encrypted.
  - Enable Kubernetes secret encryption at rest with KMS.

- **Tenant isolation**
  - At app layer: tenant ID from auth token; every DB query scoped to tenant; row-level or schema-level multi-tenancy.
  - For high-value tenants: separate DB schema or separate DB cluster, dedicated namespace, potential dedicated node group.

#### 1.4 Kubernetes & app design

- **EKS**
  - 1 prod cluster per region; optional separate clusters per environment.
  - **Node groups**:
    - `system` node group for core addons (Ingress, metrics-server, CNI, etc.).
    - `general` node groups for stateless workloads (mixed instances).
    - Optional `gpu` or `high-mem` node groups for special workloads.
  - Use **managed node groups** or **Fargate** for minimal ops where appropriate.

- **Workload structure**
  - Each app service in its own **namespace** (or group by domain), plus `tenant-<id>` namespaces if needed.
  - Pod security:
    - Pod Security Standards (restricted).
    - No root containers, read-only filesystems where possible.
  - Network policies:
    - Deny-all baseline; explicitly allow service-to-service, service-to-database traffic.

- **Scalability**
  - **Horizontal**:
    - HPA on CPU, memory, and SLI-based metrics (Requests/sec, p95 latency).
    - Cluster Autoscaler (CA) tied to node groups with proper tags.
  - **Vertical** (careful): VPA in recommend mode to inform resource tuning.

- **State & data**
  - Persistent workloads on `gp3` EBS via `StorageClass`.
  - RDS/Aurora for relational multi-tenant database.
  - ElastiCache Redis for caching / sessions / rate limiting (with per-tenant keys).

#### 1.5 Monitoring, CI/CD integration, DR hooks

- **Monitoring hooks**
  - EKS add-ons: metrics-server, CloudWatch Container Insights or Prometheus.
  - Central logging via Fluent Bit / Fluentd to CloudWatch Logs or Loki.

- **CI/CD**
  - CI builds Docker images → push to **ECR**.
  - CD via Argo CD / Flux (GitOps) or CodePipeline/GitHub Actions applying K8s manifests/Helm charts.
  - Promotion flows through `dev → stg → prod`, with progressive delivery (blue/green or canary using Argo Rollouts).

- **DR**
  - RDS cross-region read replicas.
  - S3 with cross-region replication.
  - EKS infra defined in Terraform and reproducible in secondary region.
  - Backups scheduled via AWS Backup & Velero for K8s resources/PVs.

#### 1.6 Trade-offs & justification

- **Multi-tenant DB vs per-tenant DB**:
  - **Shared schema**: lower cost/complexity; higher blast radius.
  - **Schema-per-tenant**: better isolation; more complexity in migrations.
  - **DB-per-tenant**: strongest isolation but expensive and operationally heavy.
- **Single prod EKS cluster vs many**:
  - **Single**: easier to operate, better bin-packing, simpler observability.
  - **Many**: strong isolation (esp. for enterprise tenants) but more ops overhead.
- **Service mesh**:
  - Adds mTLS, traffic splits, rich telemetry, but more complexity. Start with simple Ingress + NetworkPolicies; add mesh when scale/requirements justify.

---

### Section 2: Kubernetes Deep Dive & Troubleshooting

Incident: latency spikes, `CrashLoopBackOff` pods, `NotReady` nodes, increased error rates.

#### 2.1 Initial triage (minimize MTTR)

- **Check overall health**

```bash
# Node & pod health
kubectl get nodes -o wide
kubectl get pods -A -o wide

# Quickly inspect failing namespaces
kubectl get pods -n prod -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -n 50
```

- **Check metrics / dashboards**
  - Look at p95/p99 latency, error rates (4xx/5xx), pod restarts, node CPU/memory/disk pressure, API error rate.

#### 2.2 Investigate `NotReady` nodes

- **Describe nodes**

```bash
kubectl describe node <node-name>
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'
```

- **Hypotheses**
  - CNI failures (e.g., AWS VPC CNI out of IPs).
  - Disk pressure (logs filling, ephemeral storage).
  - Kubelet or container runtime issues.
  - Underlying EC2 issues (network, EBS, AZ impairment).

- **Immediate mitigations**
  - If node is clearly unhealthy, **cordon + drain** to move workloads:

```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

  - Check EC2 console: instance status, system logs, network status.
  - If many nodes affected in one AZ → consider **temporarily removing node group** in that AZ and rebalancing.

#### 2.3 Investigate `CrashLoopBackOff` pods

- **Get logs and events**

```bash
kubectl get pods -n prod | grep CrashLoopBackOff
kubectl describe pod <pod> -n prod
kubectl logs <pod> -n prod --previous=true
```

- **Common hypotheses**
  - App-level bug, bad config or missing secrets.
  - Resource limits too tight → OOMKilled.
  - Dependency unavailable (DB, cache).
  - Init containers failing (e.g., migration job).

- **Immediate mitigations**
  - Roll back to previous known-good Deployment:

```bash
kubectl rollout history deploy <svc> -n prod
kubectl rollout undo deploy <svc> -n prod --to-revision=<n>
```

  - Temporarily increase resources if clearly OOM:

```bash
kubectl edit deploy <svc> -n prod  # raise requests/limits + reapply properly via Git later
```

  - If due to dependency outage, redirect traffic away from failing path (feature flag, return graceful errors, use circuit breaker).

#### 2.4 Latency & error rates

- **Check live metrics**
  - p95 latency per endpoint (Prometheus/Grafana).
  - 5xx by service and error type (App/ALB/EKS).

- **Commands / checks**
  - Confirm readiness probes:

```bash
kubectl get deploy -n prod
kubectl describe deploy <svc> -n prod | egrep -A3 "Readiness|Liveness"
```

  - Validate that HPA scaled out enough and pods are actually getting scheduled:

```bash
kubectl get hpa -n prod
kubectl get pods -n prod -o wide --field-selector status.phase!=Running
kubectl describe hpa <svc> -n prod
```

- **Hypotheses**
  - Traffic spike not matched by scaling (HPA thresholds too high, slow scaleup).
  - Thundering herd due to misconfigured retries.
  - Downstream DB contention (locks, connection pool exhaustion).

- **Mitigations**
  - Temporarily **raise replicas** and/or **lower HPA thresholds**.
  - Implement or tighten **rate limiting** at API gateway/Ingress.
  - Tune DB connection pools and add caching to hot paths.

#### 2.5 Root-cause drill-down (example scenario)

Example: logs show app containers failing due to `could not connect to postgres` while nodes in one AZ are `NotReady` and RDS in another AZ is overloaded.

- Root cause chain:
  - New deployment increased connection pool per pod.
  - HPA scaled out → DB overwhelmed.
  - DB response times spiked → timeouts → CrashLoopBackOff (init/migration).
  - Pods churning caused massive log volume → filled node disk → `DiskPressure` → `NotReady`.

- Long-term fixes:
  - Cap max DB connections per tenant/service.
  - Implement backoff/retry and circuit breaking.
  - Log sampling or rate limiting; move logs off-node quickly.
  - Capacity planning for RDS; introduce read replicas/caching.

#### 2.6 Kubernetes networking failures

- **Symptoms**: timeouts between services, DNS failures, `Connection refused`.

- **Commands**
  - Check CNI and core DNS:

```bash
kubectl get pods -n kube-system -o wide
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl exec -it <pod> -n prod -- nslookup <service>.<namespace>.svc.cluster.local
kubectl exec -it <pod> -n prod -- curl http://<service>.<namespace>.svc.cluster.local:port/health
```

  - Look for NetworkPolicy misconfig:

```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy <np> -n <ns>
```

- **Mitigation**
  - Roll back recent CNI or NetworkPolicy changes.
  - If CNI plugin is broken on a subset of nodes: cordon/drain those nodes and recreate node group.

#### 2.7 Security incident: over-permissive RBAC & exposed secrets

- **Detection**
  - Alert triggered: user/service account performed unexpected actions (e.g., listing all secrets).

- **Immediate actions**
  - Revoke compromised credentials:
    - Delete associated `ServiceAccount` tokens, rotate IAM keys.
    - Revoke any exposed API tokens (GitHub, payment providers).
  - Lock down RBAC:

```bash
kubectl get clusterrolebinding -A
kubectl describe clusterrolebinding <binding>
# Remove or narrow bindings granting cluster-admin to service accounts.
```

  - Force **secrets rotation** for all impacted services.

- **Forensics**
  - Review audit logs (K8s API, CloudTrail).
  - Determine time window and blast radius (which namespaces, secrets, external systems touched).

- **Long-term remediation**
  - Enforce **least-privilege RBAC** templates, avoid `cluster-admin` for apps.
  - Policy-as-code (OPA/Gatekeeper or Kyverno) to block dangerous roles/bindings.
  - Secret scanning in repos & registries; mandatory secret management via SM/SSM.
  - Regular RBAC reviews and automated drift detection.

---

### Section 3: Infrastructure as Code & Automation (Terraform)

#### 3.1 Repo structure & modules

- **Monorepo structure**:

```text
infra/
  modules/
    vpc/
    eks/
    rds/
    redis/
    alb/
    observability/
  envs/
    dev/
      main.tf
      backend.tf
      variables.tf
    stg/
      main.tf
      backend.tf
      variables.tf
    prod/
      main.tf
      backend.tf
      variables.tf
```

- **Principles**
  - Reusable, versioned modules (tagged in Git).
  - Envs consume modules with different `tfvars` only, not copy-pasted code.
  - Keep **blast radius** small: separate state per major domain (network, app, data).

#### 3.2 State management

- **Backend**
  - S3 backend per account/region with DynamoDB table for state locking.
  - Separate S3 prefixes/state files per environment and domain:
    - `network-prod.tfstate`, `eks-prod.tfstate`, `db-prod.tfstate`, etc.

- **Access**
  - Only CI role and a small SRE group with break-glass access can modify prod state.
  - Strict bucket policies and SSE-KMS encryption.

#### 3.3 Secrets

- No secrets in `.tfvars` stored in Git.
- Use:
  - Terraform data sources for AWS Secrets Manager / SSM (read-only reference).
  - CI to inject sensitive values as environment variables or from a secure secret store.

#### 3.4 CI/CD for Terraform

- **Workflow**
  - PR opened → `terraform fmt`, `validate`, `tflint`, and `terraform plan` for target env.
  - `plan` output stored as artifact and commented on PR.
  - Merge to `main` for prod requires:
    - Approved PR.
    - Possibly a `terraform apply` **with manual approval gate**.

- **Branch/environments**
  - `feature/*` branches: plans only (no apply).
  - `dev`/`stg` can auto-apply on merge to corresponding branches/org paths.
  - `prod` applies require explicit approval (e.g., GitHub Environments or manual job).

#### 3.5 Risky Terraform change example & prevention

**Example risky change**:

- Engineer updates ASG node group:

```hcl
desired_capacity = 5
max_size        = 50  # previously 10
```

and increases pod resource requests, potentially causing massive autoscale.

**Risk**:
- Under load, Cluster Autoscaler can scale up to 50 nodes, spiking EC2 cost, API limits, and DB load.

**Prevention**:
- **Policy-as-code** for Terraform (OPA/Conftest, Terraform Cloud Policies, or infracost policies):
  - Disallow `max_size` over environment-specific thresholds without explicit waiver.
- **Code review checklist**:
  - Changes to ASGs, scaling limits, security groups, and IAM roles require senior review.
- **Automated checks**:
  - Infracost in CI to flag large cost deltas.
  - `tflint` rules for risky CIDR blocks (`0.0.0.0/0` to sensitive ports) and over-broad IAM policies.

---

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

---

### Section 5: Cloud Cost Optimization

Incident: **35% month-over-month AWS cost increase**.

#### 5.1 Investigation process

- **Data sources**
  - AWS Cost Explorer + Cost & Usage Report (CUR).
  - Group by: service, region, tag (`Environment`, `Team`, `App`, `Tenant`).

- **Steps**
  - Identify which **services** contributed most to the delta: EKS/EC2, RDS, S3, NAT, data transfer, etc.
  - Drill into **dimensions**:
    - Resource IDs (instances, clusters).
    - `kubernetes.io/cluster` and workload tags.
  - Align with **deployment history** and usage metrics.

#### 5.2 Immediate controls

- **Quick wins**
  - Identify idle or underutilized:
    - EC2/EKS nodes with <10–20% utilization for sustained periods.
    - Old RDS snapshots, unattached EBS volumes, idle NAT Gateways/ALBs.
  - Shut down non-essential dev/staging environments overnight or on schedule.
  - Reduce over-provisioned resources:
    - Downsize instance types, HPA floors, RDS instance classes (after confirming headroom).

#### 5.3 Long-term strategy

- **Governance**
  - Enforce mandatory **cost allocation tags**.
  - Monthly cost reports per team and app.
  - Budgets and alerts (e.g., 80%, 100%, 120% of expected spend).

- **Optimization**
  - **Right-size** EKS: tune requests/limits, use bin-packing friendly instance sizes.
  - Use **Savings Plans** or Reserved Instances for baseline capacity.
  - Implement **autoscaling** on all layers (EKS, RDS where appropriate).
  - Cache heavily used queries, reduce unnecessary data transfers, compress logs.

- **Engineering practices**
  - Include **Infracost** or similar in PRs to show estimated cost delta.
  - Performance tests before large rollouts to avoid over-scaling.

---

### Section 6: Disaster Recovery & Business Continuity

#### 6.1 DR strategy, RPO/RTO

- **Targets** (example for SaaS platform):
  - **RPO** (data loss window): 5–15 minutes for core DB (point-in-time with frequent binlog backups).
  - **RTO** (time to restore service): ≤ 1 hour for full region outage.

- **Approach**
  - **Active–Passive multi-region** for initial phases:
    - Primary region: full capacity, active traffic.
    - Secondary: warm or pilot-light (scaled-down EKS + RDS read replica or snapshot-based).
  - Data replication:
    - RDS cross-region read replica (or Aurora Global Database).
    - S3 with cross-region replication.
    - State/config in Git + Terraform + GitOps.

#### 6.2 Backups

- **DB**
  - Automated backups, PITR, weekly/monthly snapshots.
  - Cross-region snapshot copy or read-replica.

- **K8s & configuration**
  - `git` as source of truth for manifests and Terraform.
  - Velero (or similar) for K8s resource + PV backups to cross-region S3.

- **App & artifacts**
  - Docker images in ECR, replicated or pullable from secondary region.

#### 6.3 Regional failover flow

Simulated full region outage (Region A down; fail in Region B):

1. **Detect**: CloudWatch synthetics + Route 53 health checks detect Region A down.
2. **Decision**: On-call SRE + incident commander decide to failover.
3. **Activate infra in Region B**:
   - If not already active:
     - Run Terraform to ensure VPC/EKS/RDS (or use pre-provisioned cluster).
     - Promote RDS replica in Region B to primary.
4. **Switch traffic**:
   - Update Route 53 records (latency or failover routing) to direct traffic to Region B ALB.
   - CloudFront origins updated / multi-origin failover configured in advance.
5. **Validate**:
   - Run smoke tests; verify core flows.
   - Communicate status to customers.

After primary region recovers:
- Decide whether to **fail back** or treat Region B as new primary.
- Sync data carefully (no split-brain).

---

### Section 7: Documentation & Leadership

#### 7.1 Incident runbook outline

- **Title & scope**
  - e.g., `API Latency and Errors – Prod Runbook`.

- **Sections**
  - **Symptoms**: what alerts trigger this runbook; user-facing signals.
  - **Quick checks**:
    - Links to:
      - Grafana dashboards.
      - Kibana/CloudWatch Log insights queries.
      - `kubectl` snippets for common checks.
  - **Decision tree**:
    - If nodes `NotReady` → follow Node Health subsection.
    - If only 1 service affected → follow Service Health subsection.
  - **Standard actions**:
    - Steps for rollback, scaling, known mitigations.
  - **Escalation**:
    - When to bring in DB team, networking, security.
  - **Post-incident steps**:
    - Ensure tickets created and postmortem scheduled.

#### 7.2 On-call escalation guide outline

- **Purpose**
  - Define who responds, in what order, and expectations.

- **Contents**
  - **Roles**:
    - Primary on-call, secondary on-call, incident commander, comms lead.
  - **Contact routes**:
    - PagerDuty/ops tool, Slack channels, phone/SMS escalation.
  - **Severities & response times**:
    - SEV-1: P0 global outage, 24/7 paging, 5-min ack.
    - SEV-2: Major partial outage, 15-min ack, business hours priority.
  - **Escalation paths**:
    - When primary doesn’t respond → auto page secondary after N minutes.
    - When engineering blocked → involve vendor support or infra leadership.
  - **Handovers**:
    - Shift-change procedures; status handoff templates.

#### 7.3 Mentoring & communication

- **Mentoring junior engineers**
  - Pair them on **incident retros** so they see reasoning, not just commands.
  - Encourage them to write or update **runbooks** after incidents; review together.
  - Use **post-incident “learning sessions”** to walk through timelines and tools.
  - Give them contained ownership: e.g., “You own logging improvements for service X.”

- **Communicating incidents**

  - **To executives / non-technical stakeholders**:
    - Focus on **impact, timeline, and mitigation**:
      - “From 10:02–10:18 UTC, ~30% of requests failed for EU customers due to a database capacity issue. We rolled back a configuration change and increased capacity. We’re now implementing safeguards so this config cannot be changed without review.”
    - Avoid deep technical jargon; use visuals where helpful.
  - **To engineering teams**:
    - Provide detailed **technical analysis**:
      - Metrics graphs, specific root cause (e.g., HPA + DB pool misconfiguration), exact fixes.
    - Link to runbooks, PRs, and follow-up work items.
    - Encourage feedback: what tooling or docs would have made this easier?

---

This document can be exported directly as Markdown or converted to PDF for submission.

