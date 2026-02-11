### Section 1: Cloud Architecture & Design (AWS EKS Multitenant SaaS)

#### 1.1 High-level architecture

**Goals**: public-facing, secure multitenancy, autoscaling, strong isolation, observable, DR-ready.

Conceptual diagram (simplified):


![AWS EKS Architecture](AWS cloud architecture..png)

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
