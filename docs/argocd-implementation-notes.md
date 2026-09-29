# ArgoCD Implementation Notes

Consolidated reference for how ArgoCD was set up on the real EKS cluster, and every issue hit + fix along the way.

---

## 1. Installation

**Approach:** version-pinned Helm install with a custom `values.yaml`, not a bare default install — matching the "enterprise-lite" discipline applied elsewhere in this project (pinned versions, explicit resource limits, declarative config over imperative CLI/UI actions).

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd -n argocd --create-namespace --version 10.9.2 -f values.yaml
```

- **Chart version pinned:** `10.9.2` (deploys ArgoCD `v3.0.0` under the hood) — checked against Artifact Hub rather than assumed, same discipline as pinning Terraform module versions.
- **Dedicated namespace:** `argocd`, created via `--create-namespace`, keeping platform tooling separate from application workloads (same pattern as `ingress-nginx`, `kube-system`).

## 2. `values.yaml`

```yaml
server:
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"

repoServer:
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"

controller:
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
```

**Why:** resource requests/limits on ArgoCD's own components, matching the same discipline applied to the app's own workloads — not required for a demo, but a deliberate signal of operational maturity.

**Key naming gotcha:** the correct key is `repoServer` (camelCase) — an initial attempt used `repo-server` (hyphenated), which is wrong for this chart. Verified against the chart's actual documented values schema before proceeding, rather than assuming.

## 3. Accessing the UI

No Ingress/TLS set up for the ArgoCD UI in this phase (explicitly deferred to Phase 2 — "Ingress + TLS for the ArgoCD UI, vs port-forward"). Accessed via port-forward for now:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Retrieve the initial admin password (Windows/PowerShell — the Linux `base64 -d` command doesn't exist natively):

```powershell
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}")))
```

Login: username `admin`, password from the command above.

## 4. The `Application` resource (connecting ArgoCD to the app)

Written declaratively (not created via UI/CLI imperatively), applied directly:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: voting-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/likithreddy-ops/eks-platform-manifests.git
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Key fields:**
- `path: .` — the Helm chart's `Chart.yaml` sits at the repo root, not in a subfolder
- `destination.namespace: default` — matches the (deliberately simple, undivided) namespace choice used throughout this project
- `syncPolicy.automated` — `prune: true` removes resources no longer in Git; `selfHeal: true` reverts manual cluster drift back to match Git — the actual "GitOps" behavior, not just one-time deployment

```bash
kubectl apply -f application.yaml
```

## 5. Issues hit during ArgoCD rollout, in order

### 5.1 — "Too many pods" scheduling failures
Once ArgoCD's own pods (server, repo-server, controller, dex, redis) joined the cluster on top of the 6 app pods, new pods started failing:
```
0/2 nodes are available: 1 Insufficient memory, 2 Too many pods.
```
**Root cause:** `t3.small` has a hard ceiling on pods-per-node driven by ENI/IP capacity — not raw CPU/memory. `kubectl describe node` showed plenty of CPU/memory headroom but hit the pod-count limit regardless.

**Fix:** scaled the node group from 1 → 3 nodes. Re-confirmed that `desired_size` changes must go through `aws eks update-nodegroup-config` (AWS CLI) — `terraform apply` shows "No changes" even after editing `desired_size` in the `.tf` file, because the AWS provider deliberately ignores drift on this specific field (to avoid fighting an external autoscaler like Cluster Autoscaler).

### 5.2 — Postgres PVC stuck `Pending`
Worked fine on minikube; failed on real EKS with:
```
no persistent volumes available for this claim and no storage class is set
```
**Root cause:** unlike minikube (built-in default provisioner), a Terraform-created EKS cluster has **no default StorageClass and no EBS CSI driver installed by default** — same pattern as the missing `vpc-cni`/`coredns`/`kube-proxy` addons from initial cluster bring-up.

**Fix:**
- Added `aws-ebs-csi-driver` to the `addons` block
- Required its own IRSA (IAM Roles for Service Accounts) module:
  ```hcl
  module "ebs_csi_driver_irsa" {
    source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
    version = "~> 5.20"

    role_name             = "ebs-csi-driver"   # NOT "name" - verified against actual module docs after an initial guess failed
    attach_ebs_csi_policy = true

    oidc_providers = {
      this = {
        provider_arn               = module.eks.oidc_provider_arn
        namespace_service_accounts = ["kube-system:ebs-csi-controller-sa"]
      }
    }
  }
  ```
  ```hcl
  addons = {
    vpc-cni    = {}
    coredns    = {}
    kube-proxy = {}
    aws-ebs-csi-driver = {
      service_account_role_arn = module.ebs_csi_driver_irsa.iam_role_arn
    }
  }
  ```
- Annotated the existing `gp2` StorageClass as cluster default:
  ```bash
  kubectl annotate storageclass gp2 storageclass.kubernetes.io/is-default-class=true
  ```
  (`kubectl patch` with inline JSON kept failing due to PowerShell quote-mangling; `annotate` avoided the problem entirely by not needing JSON.)

### 5.3 — Postgres crash-looping on `initdb`, even after the PVC bound
```
initdb: directory "/var/lib/postgresql/data" exists but is not empty
It contains a lost+found directory
```
**Root cause:** AWS EBS automatically creates a `lost+found` directory at the root of every fresh volume. `initdb` refuses to initialize unless the directory is completely empty — even for just this one filesystem artifact. Didn't show up on minikube, since its local storage provisioner doesn't create this directory.

**Fix:** added `PGDATA=/var/lib/postgresql/data/pgdata` (pointing Postgres at a *subdirectory* of the mount, not the mount root) — matching the official reference manifests, a detail noted early in the project but not carried into the actual manifest until this forced the issue.

### 5.4 — Kubernetes rejected the StatefulSet update that added `PGDATA`
```
StatefulSet.apps "db" is invalid: ... updates to statefulset spec for fields
other than 'replicas', 'ordinals', 'template', ... are forbidden
```
Even though the changed field (`env`, under `template`) should have been allowed, the patch as a whole was rejected.

**Fix:** deleted the StatefulSet with `--cascade=orphan` (preserving the pod/PVC temporarily) and let ArgoCD recreate it fresh from the current manifest — sidestepping the in-place-update restriction. A known, common StatefulSet friction point, not specific to this project.

### 5.5 — Existing pod didn't pick up the fix even after the StatefulSet was recreated
StatefulSets don't automatically recreate already-existing pods just because the StatefulSet resource itself was recreated.

**Fix:** explicitly deleted the pod (`kubectl delete pod db-0`). When the same `lost+found` error still appeared (because the *volume itself* still had the old artifact at its root, independent of the new `PGDATA` subdirectory setting), also deleted the PVC (`kubectl delete pvc db-data-db-0`) to force a genuinely fresh EBS volume — safe here since Postgres had never successfully started even once.

## 6. Result

Full GitOps pipeline confirmed: `git push` → ArgoCD auto-sync (`Synced`, `Healthy`) → live deployment on real EKS. Cast a vote through Vote's UI (via port-forward) and confirmed it appeared on Result's page — the entire chain (Vote → Redis → Worker → Postgres → Result), deployed declaratively via ArgoCD rather than manual `kubectl apply`, working end-to-end on real AWS infrastructure.

## 7. Deferred to Phase 2 (ArgoCD-specific)

- App of Apps pattern (self-managing ArgoCD's own config via GitOps)
- SSO/Okta integration for the ArgoCD UI login
- HA mode (multi-replica ArgoCD components)
- Notifications/webhooks (Slack/PagerDuty on sync failure)
- Ingress + TLS for the ArgoCD UI (replacing port-forward)
- Custom Resource Health checks
- RBAC policy customization within ArgoCD
