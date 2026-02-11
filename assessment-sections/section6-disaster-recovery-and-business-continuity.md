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
