# eks-platform-gitops

GitOps and observability configuration for the platform — ArgoCD (continuous deployment) and Prometheus/Grafana (monitoring) — managed as its own repo, separate from raw infrastructure ([eks-platform-infra](https://github.com/likithreddy-ops/eks-platform-infra)) and application code ([eks-platform-manifests](https://github.com/likithreddy-ops/eks-platform-manifests)).

Kept as a dedicated repo specifically to support future automation: a planned GitHub Actions workflow to install/upgrade ArgoCD and monitoring, and to eventually support the "App of Apps" GitOps pattern.

---

## Contents

```
argocd/
  values.yaml       # ArgoCD Helm chart configuration (resource limits on server/repoServer/controller)
  application.yaml  # Declarative Application resource pointing at eks-platform-manifests
monitoring/
  values.yaml       # kube-prometheus-stack Helm chart configuration
```

## ArgoCD

Version-pinned Helm install (`10.9.2`), custom `values.yaml` with explicit resource requests/limits on `server`, `repoServer`, and `controller` — the same discipline applied to the application's own workloads, not left at chart defaults.

```bash
cd argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace --version 10.9.2 -f values.yaml
kubectl apply -f application.yaml
```

The `application.yaml` `Application` resource uses `syncPolicy.automated` (`prune: true`, `selfHeal: true`) — real GitOps behavior, not just a one-time deploy: it continuously reconciles the live cluster against what's committed in `eks-platform-manifests`.

## Monitoring

`kube-prometheus-stack` (Prometheus + Grafana + Alertmanager + kube-state-metrics + node-exporter), version-pinned (`91.5.0`), also with a custom `values.yaml`.

```bash
cd monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace --version 91.5.0 -f values.yaml
```

Access Grafana:
```bash
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
```

## Real Issues Hit & Fixed

Full detail in `docs/argocd-implementation-notes.md` and `docs/monitoring-implementation-notes.md`:

1. **"Too many pods" scheduling failures — twice.** `t3.small`'s hard ceiling on pods-per-node (ENI/IP capacity, not CPU/memory) was exceeded first when ArgoCD's own pods landed alongside the app, then again when the full Prometheus stack was added. Fixed by scaling the node group (1 → 3 → 4 nodes over the course of the build), confirmed via the AWS Console that nodes auto-spread evenly across both AZs.
2. **A stuck ArgoCD `Application` sync** caused by a StatefulSet update Kubernetes rejected outright (most in-place spec field changes are forbidden) — resolved by deleting and letting ArgoCD recreate the resource.
3. **A stuck Prometheus `admission-patch` Job** referencing a ServiceAccount that didn't exist yet — a Helm chart hook-ordering race condition; the Job was non-critical (webhook validation only) and was deleted rather than chased further.
4. **Grafana repeatedly `OOMKilled`.** Logs showed a completely healthy startup — the *absence* of any error was itself the diagnostic clue. Confirmed via `lastState.terminated.reason: OOMKilled` (exit code `137`) that the original `256Mi` memory limit was too tight for this Grafana version (`13.2.2`) plus its `k8s-sidecar` provisioning containers. Fixed by raising the limit to `512Mi`.
5. **`kubectl port-forward` reliability** — not a bug, a known limitation of the tool (drops on terminal focus loss or target-pod crashes). Documented a simple PowerShell retry-loop workaround; the real fix (Ingress + TLS) is a planned enhancement.

## Verified Working

- ArgoCD: `Application` shows `Synced` / `Healthy`; full voting-app pipeline confirmed working end-to-end on real EKS after a from-scratch rebuild, proving the whole sequence (infra → ArgoCD → monitoring) is genuinely repeatable, not a one-off.
- Grafana: live dashboards (e.g., "Kubernetes / Compute Resources / Cluster") showing real per-namespace CPU/memory usage across `default`, `kube-system`, `argocd`, and `monitoring`.

## Deferred / Future Enhancements

**ArgoCD:**
- App of Apps pattern (self-managing ArgoCD's own config via GitOps)
- SSO/Okta integration for the UI login; a fixed/known admin password via `values.yaml`
- HA mode (multi-replica components); Ingress + TLS (replacing port-forward)
- Notifications/webhooks (Slack/PagerDuty on sync failure)
- Replacing the app's managed node group with a custom resource + explicit `depends_on` on add-ons, to eliminate the pod-scheduling race condition at its root (infra-side change, tracked in `eks-platform-infra`)

**Monitoring:**
- Ingress + TLS for Grafana; a fixed/known admin password
- Alertmanager routing to Slack/PagerDuty; custom PrometheusRule/alert definitions
- Defined SLOs/SLIs and an error budget for the voting app, measured via this existing Prometheus/Grafana stack
- A GitHub Actions workflow to automate the installs in this repo
