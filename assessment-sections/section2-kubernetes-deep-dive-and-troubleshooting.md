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
