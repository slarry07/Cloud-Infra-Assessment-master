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
