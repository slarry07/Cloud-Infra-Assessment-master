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
