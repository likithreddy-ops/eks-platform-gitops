# Monitoring (Prometheus + Grafana) Implementation Notes

Consolidated reference for how the observability stack was set up on the real EKS cluster, and every issue hit + fix along the way.

---

## 1. What was installed

**Chart:** `kube-prometheus-stack` — a combined chart that bundles:
- **Prometheus** — metrics collection/storage engine
- **Grafana** — dashboard/visualization layer
- **Alertmanager** — alerting (installed, no custom alert rules configured yet)
- **Node Exporter** — per-node hardware/OS metrics (runs as a DaemonSet, one pod per node)
- **kube-state-metrics** — exposes Kubernetes object state (pod counts, deployment status, etc.) as metrics
- **Prometheus Operator** — manages Prometheus/Alertmanager configuration via CRDs

One chart, one install, the full observability pipeline working together with pre-built dashboards included.

**Separately, not part of this chart:** Metrics Server — a lighter-weight component specifically needed to feed CPU/memory data to `kubectl top` and HPA. Installed as its own step, after this stack.

## 2. Installation

Version-pinned, custom `values.yaml` — same "enterprise-lite" discipline as the ArgoCD setup (pin versions, explicit resource limits, don't just take chart defaults).

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm search repo prometheus-community/kube-prometheus-stack --versions
```

Confirmed current version live via `helm search` rather than trusting a web search result (which showed an older `81.x` line — the actual current version was `91.5.0`, deploying Prometheus `v0.94.0`).

```bash
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace --version 91.5.0 -f values.yaml
```

## 3. `values.yaml`

```yaml
prometheus:
  prometheusSpec:
    resources:
      requests:
        cpu: "100m"
        memory: "256Mi"
      limits:
        cpu: "250m"
        memory: "512Mi"

grafana:
  resources:
    requests:
      cpu: "50m"
      memory: "256Mi"
    limits:
      cpu: "200m"
      memory: "512Mi"

alertmanager:
  alertmanagerSpec:
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "128Mi"
```

**Structural note:** `prometheus` and `alertmanager` need an extra nested level (`prometheusSpec` / `alertmanagerSpec`) before `resources`, because this chart manages them via Prometheus Operator CRDs rather than directly — different nesting pattern from `grafana`, which is more directly structured (closer to how ArgoCD's `values.yaml` worked).

**Grafana's memory limit was revised upward** after hitting a real issue — see 4.3 below. Original attempt used `128Mi` request / `256Mi` limit; final working values shown above.

## 4. Issues hit during rollout, in order

### 4.1 — "Too many pods" scheduling failures (again)
Same root cause as the ArgoCD rollout the day before: `t3.small`'s hard ceiling on pods-per-node (driven by ENI/IP capacity, not CPU/memory) got exceeded once this stack's pods (Prometheus, Grafana, Alertmanager, kube-state-metrics, Prometheus Operator, one node-exporter per node) landed on top of everything already running (app pods + ArgoCD + EBS CSI driver).

**Fix:** scaled the node group from 3 → 4 nodes:
```bash
# max_size bumped in eks.tf first, then terraform apply
aws eks update-nodegroup-config --cluster-name platform-eng-cluster --nodegroup-name <name> --scaling-config minSize=1,maxSize=4,desiredSize=4 --region us-east-1
```
Confirmed nodes auto-spread evenly across both AZs (2/2 split) via the AWS Console — Auto Scaling Group balances instances across all subnets/AZs it has access to, without any extra configuration needed.

### 4.2 — Stuck `admission-patch` Job (missing ServiceAccount)
A one-time Helm hook Job (`monitoring-kube-prometheus-admission-patch`) stayed `ContainerCreating`/`Pending` for 14+ minutes:
```
FailedMount: failed to fetch token: serviceaccounts "monitoring-kube-prometheus-admission" not found
```
**Root cause:** a Helm chart hook-ordering race condition — the Job's pod appears to have been scheduled before its own required ServiceAccount had been created by the same install (similar in category to the ArgoCD/addon race condition from the day before, though this one wasn't chased to a definitive root cause given time already invested).

**Fix:** confirmed the ServiceAccount genuinely didn't exist (not just a stale retry), then deleted the stuck Job entirely — it only handles webhook admission validation for custom PrometheusRule syntax, non-critical to core Prometheus/Grafana functionality:
```bash
kubectl delete job monitoring-kube-prometheus-admission-patch -n monitoring
```

### 4.3 — Grafana repeatedly OOMKilled
Grafana crash-looped (4+ restarts), breaking `kubectl port-forward` mid-session every time. Logs showed completely normal startup activity (search index building, usage stats) — no visible errors — which was itself the clue that this wasn't an application-level failure.

**Diagnosis:**
```bash
kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana -o jsonpath="{.items[0].status.containerStatuses[0].lastState}"
# -> {"terminated":{"exitCode":137,"reason":"OOMKilled",...}}
```
Confirmed definitively: exit code `137` + `OOMKilled`. The original `256Mi` memory limit was too tight for this chart's Grafana version (`13.2.2`, notably heavier than older Grafana releases) plus its accompanying `k8s-sidecar` containers for dashboard/datasource provisioning.

**Fix:** raised Grafana's memory to `256Mi` request / `512Mi` limit, then:
```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack -n monitoring --version 91.5.0 -f values.yaml
```
Restarts stopped; pod stabilized.

## 5. Accessing Grafana

No Ingress/TLS set up yet (same Phase 2 deferral as ArgoCD's UI). Accessed via port-forward:
```bash
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
```
Retrieve the actual admin password (the chart's commonly-cited default `prom-operator` did not work in this install — the chart generated its own random password instead):
```powershell
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}")))
```
Username: `admin`

**Note on port-forward reliability:** `kubectl port-forward` is not a persistent connection — it drops on terminal focus loss, idling, or (as happened here) the target pod crashing. Not a bug, just a limitation of the tool; a real fix is proper Ingress (Phase 2 item), or a simple PowerShell retry loop in the meantime:
```powershell
while ($true) { kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80; Start-Sleep -Seconds 2 }
```

## 6. Result

Verified real, live dashboards (e.g., "Kubernetes / Compute Resources / Cluster") showing actual cluster data: per-namespace CPU/memory utilization and quotas across `default`, `kube-system`, `argocd`, and `monitoring` namespaces, live CPU usage graphs over time. Confirms the full pipeline — Prometheus scraping real metrics, Grafana querying and rendering them — working correctly on real AWS infrastructure.

## 7. Still pending

- Metrics Server install (separate, lightweight component, needed specifically for HPA)
- Live HPA demo (load test, watch Vote's replicas scale)
- Push `monitoring/values.yaml` to the `eks-platform-gitops` repo

## 8. Cost note

Running cost with the full stack (EKS control plane + 4× t3.small nodes + single NAT Gateway) is approximately **$0.23/hour**. `terraform destroy -target="module.eks" -target="module.vpc"` at the end of each session, `terraform apply` to rebuild — full rebuild sequence documented in the post-destroy-rebuild-runbook.

## 9. Deferred to Phase 2 (monitoring-specific)

- Ingress + TLS for the Grafana UI (replacing port-forward)
- Fixed/known Grafana admin password via `values.yaml` (avoid re-decoding a random one each fresh install)
- Alertmanager routing to Slack/PagerDuty (already listed under the broader Phase 2 plan)
- Custom PrometheusRule / alert definitions (the admission-patch webhook this would need was deleted rather than fixed — would need revisiting if custom rules are added later)
- Right-size node/instance strategy (this is now the third time the `t3.small` pod-count ceiling has been hit — a `t3.medium`-based, fewer-larger-nodes approach is a legitimate architectural alternative worth exploring)
