# Answer Key

*Step-by-step details for when you're stuck. Try to figure things out on your own first — the friction is where the learning happens.*

---

## Setting the Stage

### External Feeds

**Library → External Feeds → Add Feed:**
- Feed type: Docker Container Registry
- Name: `Docker Hub`
- URL: `https://registry-1.docker.io`
- No credentials needed for public images at this stage

Do this in both the **Payments** and **Merchants** spaces.

### RBAC

Go to **Configuration → Teams** to create teams and assign users with roles.

Key role distinctions:
- **Deployment Creator** — can create releases but not deploy them
- **Project Deployer** — can deploy releases to environments
- Use environment scoping on roles to restrict who can deploy where (e.g., Alex can deploy to Dev/Staging but not Production)
- **Runbook Producer/Consumer** roles control runbook access separately from deployment permissions

### K8s Agents

Go to **Infrastructure → Deployment Targets → Add Deployment Target → Kubernetes Agent**. The wizard walks you through everything and generates the Helm command to run on each cluster context.

---

## Chapter 1: payments-api

### Helm Step Configuration

| Setting | Value |
|---------|-------|
| Target tags | `payments-k8s` |
| Chart source | Git Repository |
| Chart path | `charts/payments-api` |
| Values file 1 | `charts/payments-api/values.yaml` |
| Values file 2 | `charts/payments-api/values-#{Octopus.Environment.Name \| ToLower}.yaml` |
| Helm release name | `payments-api` |
| Namespace | `#{Namespace}` |

The key trick is `#{Octopus.Environment.Name | ToLower}` — this dynamically picks `values-development.yaml`, `values-staging.yaml`, or `values-production.yaml` based on the target environment.

### Project Variables

| Variable | Value | Scope |
|----------|-------|-------|
| `Namespace` | `payments-dev` | Development |
| `Namespace` | `payments-staging` | Staging |
| `Namespace` | `payments-prod` | Production |

### Production Approval Gate

Add a **Manual Intervention** step to the deployment process and scope it to the **Production** environment only (under Conditions → Environments). Place it BEFORE the Helm step. Dev and staging deploys skip it automatically.

---

## Chapter 2: fraud-detector & Runbooks

### fraud-detector Helm Step

Same pattern as payments-api:

| Setting | Value |
|---------|-------|
| Target tags | `payments-k8s` |
| Chart source | Git Repository |
| Chart path | `charts/fraud-detector` |
| Values file 1 | `charts/fraud-detector/values.yaml` |
| Values file 2 | `charts/fraud-detector/values-#{Octopus.Environment.Name \| ToLower}.yaml` |
| Helm release name | `fraud-detector` |
| Namespace | `#{Namespace}` |

### Restart Service Runbook

Create a runbook on the fraud-detector project. Add a **Run a Script** step with target tag `payments-k8s`:

```bash
echo "Restarting fraud-detector in #{Octopus.Environment.Name}..."
kubectl rollout restart deployment/fraud-detector -n #{Namespace}
kubectl rollout status deployment/fraud-detector -n #{Namespace} --timeout=120s
echo ""
echo "Pod status after restart:"
kubectl get pods -n #{Namespace} -l app=fraud-detector
```

### Provision Namespace Runbook

Create a `cluster-ops` project in the **Platform** space (no deployment process needed). Add a "Provision Namespace" runbook.

**Prompted Variables** — go to the project's **Variables → Project Variables** and create these with **"Prompt for value"** checked:

| Variable | Label | Type |
|----------|-------|------|
| `NamespaceName` | Namespace Name | Single-line text |
| `TeamLabel` | Team Label | Single-line text |

Add a **Run a Script** step with target tag `platform-k8s`:

```bash
echo "Provisioning namespace #{NamespaceName} for team #{TeamLabel}..."

kubectl create namespace #{NamespaceName} --dry-run=client -o yaml | \
  kubectl label --local -f - \
    team=#{TeamLabel} \
    managed-by=octopus \
    -o yaml | kubectl apply -f -

echo "✅ Namespace #{NamespaceName} is ready."
kubectl get namespace #{NamespaceName} --show-labels
```

---

## Chapter 3: kyc-service (Raw YAML)

### Step Type

The step you're looking for is **Deploy Kubernetes YAML** (not Helm). Point it at Git, path: `manifests/kyc-service/deployment.yaml`.

### Variables

Look at the YAML file — every `#{...}` placeholder needs a corresponding project variable:

| Variable | Dev | Staging | Prod |
|----------|-----|---------|------|
| `Namespace` | `kyc-dev` | `kyc-staging` | `kyc-prod` |
| `Replicas` | `1` | `2` | `2` |
| `LogLevel` | `debug` | `info` | `warn` |
| `DocumentStorageUrl` | `s3://finpay-dev-kyc-docs` | `s3://finpay-staging-kyc-docs` | `s3://finpay-prod-kyc-docs` |

`KafkaBrokers` is NOT in this table — it comes from the **Common Config library variable set** (included via **Variables → Library Sets**). This is the whole point: shared infrastructure config lives in one place, not duplicated across every project.

### Compliance Gate

Manual Intervention step scoped to **Staging AND Production** (not just prod). Place it before the deploy step.

---

## Chapter 4: Tenants & Config-as-Code

### Tenant Variable Templates

Tenant variables in Octopus work through a **template** system: you define templates (the shape), then each tenant fills in their own values. There are two places to define templates:

**1. Project Templates** (project-specific variables):

Go to the merchant-portal project → **Tenant Variables** → **Project Templates** tab → **Add project template**:

| Variable name | Label | Control type |
|---------------|-------|-------------|
| `Namespace` | Kubernetes Namespace | Single-line text box |

This goes on the project because namespace naming is project-specific — `merchant-portal` uses `merchant-acme-dev`, but a different tenanted project would use its own convention.

**2. Common Variable Templates** (shared tenant identity):

Create a **Library Variable Set** called **"Tenant Config"** → go to the **Variable Templates** tab (not regular Variables) → Add Templates:

| Variable name | Label | Control type |
|---------------|-------|-------------|
| `BrandColor` | Brand Color | Single-line text box |
| `DataRegion` | Data Region | Single-line text box |

These go on a Library Variable Set because they're tenant-level facts that any tenanted project would need. Include this library set on the merchant-portal project (Variables → Library Sets).

### Tenant-Aware Deployment Targets

K8s Agents default to **"Exclude from tenanted deployments"** — tenanted deployments won't find them. You need to change this AND associate tenants with the target. The cleanest approach uses tenant tags:

1. **Create a Tag Set:** Library → Tenant Tag Sets → create `Hosting` (scope: Tenants) with a tag `Shared-K8s`
2. **Tag both tenants:** Tenants → Acme Corp → add the `Hosting/Shared-K8s` tag. Repeat for EuroShop.
3. **Update the K8s Agents:** Infrastructure → `merchants-dev` → Tenanted Deployments → set to **"Include in both tenanted and untenanted deployments"** → Associated Tenants → add the `Hosting/Shared-K8s` tenant tag. Repeat for `merchants-prod`.

> Why tags instead of individual tenants? When EuroShop's competitor signs up next month, you just tag the new tenant — no target reconfiguration needed.

### Enable Multi-Tenancy

Go to the merchant-portal project → **Settings → Multi-tenancy** → set to **"Allow deployments with or without a tenant"**. Octopus can also auto-enable this when you connect a tenant, but setting it explicitly avoids confusion.

### Connect Tenants to the Project

Go to **Tenants → Acme Corp → Connected Projects** → add `merchant-portal` → select **all 3 environments** (Development, Staging, Production). Repeat for EuroShop.

> The variable inputs only appear on the tenant's Variables page after the tenant is connected to environments.

### Fill In Tenant Variable Values

Go to the merchant-portal project → **Tenant Variables**. You'll see tabs for:
- **Project Templates** — `Namespace` (edit inline per tenant/environment)
- **Common Templates** — `BrandColor` and `DataRegion` from Tenant Config library set (edit inline per tenant)

**Acme Corp:**
| Variable | Value | Environment |
|----------|-------|-------------|
| `Namespace` | `merchant-acme-dev` | Development |
| `Namespace` | `merchant-acme-staging` | Staging |
| `Namespace` | `merchant-acme-prod` | Production |
| `BrandColor` | `#FF6600` | *(all)* |
| `DataRegion` | `us-east-1` | *(all)* |

**EuroShop:**
| Variable | Value | Environment |
|----------|-------|-------------|
| `Namespace` | `merchant-euro-dev` | Development |
| `Namespace` | `merchant-euro-staging` | Staging |
| `Namespace` | `merchant-euro-prod` | Production |
| `BrandColor` | `#003399` | *(all)* |
| `DataRegion` | `eu-west-1` | *(all)* |

### Helm Release Name (Avoiding Collisions)

The release name must include the tenant to avoid two tenants colliding on the same Helm release:

```
merchant-portal-#{Octopus.Deployment.Tenant.Name | ToLower | Replace " " "-"}
```

This produces `merchant-portal-acme-corp` and `merchant-portal-euroshop`.

### Multi-Tenancy Setting

In **Settings → Multi-tenancy**, change to "Allow deployments with or without a tenant."

### Injecting Tenant Variables into Helm

The `BrandColor` and `DataRegion` variables are tenant-specific — they can't live in the static per-environment values files. Add a **Key values** (or Inline YAML) template values source with these overrides:

```
env.BRAND_COLOR=#{BrandColor}
env.DATA_REGION=#{DataRegion}
```

> ⚠️ **Template Values ordering matters.** The Octopus Helm step passes all template values sources as `-f` (values files) — there's no `--set`. Sources **higher in the list take higher precedence**. If your Key values entry is below the chart's values files, the chart defaults will override your values. Use the **Reorder** button to put your overrides **above** the chart's values files.

After deploying both tenants, verify the values are actually different:

```bash
kubectl --context kind-finpay-dev exec deploy/merchant-portal-acme-corp -n merchant-acme-dev -- env | grep -E "BRAND|REGION"
kubectl --context kind-finpay-dev exec deploy/merchant-portal-euroshop -n merchant-euro-dev -- env | grep -E "BRAND|REGION"
```

### Config-as-Code

When creating the `merchant-portal` project, there's an option to store the project configuration in a Git repository instead of the Octopus database. Choose your finpay-deploy fork.

Once enabled, the deployment process and settings are represented as `.ocl` files committed to the repo. You can:
- Switch branches in the Octopus UI to work on a different version of the process
- Make changes on a feature branch, test them, then merge via PR
- Review deployment process changes alongside code changes in the same PR

---

## Chapter 5: Auto-Deploy on Image Push

### 1. Update Docker Hub Feed Credentials

**Library → External Feeds** → edit the Docker Hub feed you created earlier. Add your Docker Hub credentials so Octopus can query your private repository.

### 2. Add Container Image Reference

In the payments-api project's Helm deploy step, add a **container image package reference**. This tells Octopus to track `<your-username>/finpay-payments-api` from the Docker Hub feed. The referenced package version becomes available as an Octopus variable.

### 3. Override Helm Image Values

Add a **Key values** (or Inline YAML) template values source with:

```
image.repository=<your-username>/finpay-payments-api
image.tag=#{Octopus.Action.Package[finpay-payments-api].PackageVersion}
```

> ⚠️ **Ordering matters.** The Helm step has no `--set` — all template values sources are passed as `-f` (values files). Sources **higher in the list take higher precedence**. Reorder your image overrides **above** the chart's values files, or the chart defaults (`image.repository: nginx`) will win.

### 4. Create Auto-Release Trigger

**Project → Triggers** → create a trigger that watches for new container image versions and auto-creates a release.

### Release Versioning

**Settings → Release Versioning** → select "Use the version number from an included package" → choose the `finpay-payments-api` package reference. Now when the trigger creates a release from image tag `1.2.0`, the release itself is versioned `1.2.0`. This makes the dashboard immediately readable — you can tell which image version is running in each environment at a glance.

The other options: "Generate version numbers using a template" (useful for date-based or custom patterns like `#{Octopus.Date.Year}.#{Octopus.Date.Month}.i`) and the default auto-increment (0.0.1, 0.0.2... — meaningless for container workflows).

### Channels / Hotfix Path

Create a second channel called "Hotfix" and assign it the Hotfix lifecycle. In the channel's version rules, add a package version rule for the `finpay-payments-api` package with pre-release tag regex `^hotfix`. This matches SemVer pre-release tags like `1.3.0-hotfix` — the regex runs against just the tag portion (`hotfix`), not the full version string.

> ⚠️ **One trigger per channel.** Auto-release triggers are scoped to a single channel. You'll need a **separate trigger** for the Hotfix channel — the Default channel's trigger won't create releases for it.

---

## Chapter 7: ArgoCD Integration

### ArgoCD Gateway

Go to **Infrastructure → Argo CD Instances → Add Argo CD Instance** in the Payments space.

### ArgoCD Application Annotations

Add these annotations to each Application in `argocd-manifests/applications.yaml`:

```yaml
# payments-api-dev
annotations:
  argo.octopus.com/project: payments-api-argo
  argo.octopus.com/environment: development

# payments-api-staging
annotations:
  argo.octopus.com/project: payments-api-argo
  argo.octopus.com/environment: staging
```

These tell the ArgoCD Gateway which Octopus project and environment each Application corresponds to. Without them, Octopus has no way to map ArgoCD sync status back to its own deployment pipeline.

### Private Repos — ArgoCD Credentials

If your fork is private, ArgoCD can't clone it by default. You'll see `ComparisonError: authentication required` on the Applications. Fix: add repo credentials to ArgoCD, then delete and re-apply the Applications (ArgoCD caches the failed state).

```bash
argocd repo add https://github.com/<YOUR_USERNAME>/finpay-deploy.git \
  --username <github-username> --password <github-pat>
kubectl --context kind-finpay-dev delete -f argocd-manifests/applications.yaml
kubectl --context kind-finpay-dev apply -f argocd-manifests/applications.yaml
```

### ArgoCD Project Step

The step type is called **"Update Argo CD Application Image Tags"**. It commits image tag changes to Git, which ArgoCD then detects and syncs to the cluster. The flow is: Octopus → Git commit → ArgoCD sync → cluster updated.

---

## Chapter 8: Database Migration Runbook

### Prompted Variable

Go to **Variables → Project Variables** and add `MigrationName`. Check **"Prompt for value during deployment"** — this makes the operator enter the migration name each time they run the runbook.

### Runbook Process

**Step 1 — Manual Intervention** (scoped to Staging + Production):
- Instructions: `Database migration: #{MigrationName}. Confirm this has been tested in the previous environment.`

**Step 2 — Run a Script** (target tag: `payments-k8s`):

```bash
echo "============================================"
echo "  DATABASE MIGRATION"
echo "  Service:     payments-api"
echo "  Environment: #{Octopus.Environment.Name}"
echo "  Migration:   #{MigrationName}"
echo "  Run:         #{Octopus.RunbookRun.Name}"
echo "  Time:        $(date -u '+%Y-%m-%d %H:%M:%S UTC')"
echo "============================================"

# In production this would be:
# kubectl run migration-#{MigrationName} \
#   --image=finpay/payments-api:#{Octopus.Release.Number} \
#   --restart=Never -n #{Namespace} -- ./migrate.sh #{MigrationName}

echo "(Simulated) Running migration #{MigrationName}..."
sleep 2
echo "Migration complete."
```

**Step 3 — Run a Script** (verification, target tag: `payments-k8s`):

```bash
echo "Verifying payments-api health post-migration..."
POD=$(kubectl get pod -n #{Namespace} -l app=payments-api -o jsonpath='{.items[0].metadata.name}')
STATUS=$(kubectl exec -n #{Namespace} ${POD} -- curl -s -o /dev/null -w '%{http_code}' http://localhost:80/ 2>/dev/null)

if [ "${STATUS}" = "200" ]; then
  echo "✅ payments-api healthy after migration"
else
  echo "⚠️  payments-api returned HTTP ${STATUS} — investigate"
fi
```

---

## Chapter 9: Git-Driven Variables

### Enable Config-as-Code

If you haven't already enabled Config-as-Code on the `fraud-detector` project, do that first: **Settings → Version Control → Configure** and point it to your finpay-deploy fork.

### Move Variables to Git

Once CaC is enabled, project variables are stored in `.ocl` files in the `.octopus/` directory of your repo. When you edit variables through the Octopus UI, it commits the change to Git.

To move variables to Git:
1. Enable CaC (if not already done)
2. The existing variables will be exported to the `.ocl` files on the initial commit
3. From this point, you can edit variables either via the UI (which commits to Git) or directly in the `.ocl` files (which Octopus reads on next operation)

### What CAN'T Live in Git

**Sensitive variables** (marked as sensitive in Octopus) cannot be stored in Git — they stay in the Octopus database. This is by design: you don't want secrets in your Git history. If `fraud-detector` had API keys or database passwords, those would remain platform-managed.

**Library variable sets** also stay in the platform — they're shared across projects and aren't part of any single project's Git repository.

### Test a Git-Driven Change

1. Clone/pull your fork
2. Find the `.octopus/fraud-detector/` directory
3. Edit a variable value in the `.ocl` file (e.g., change a log level)
4. Commit and push
5. Create a new release in Octopus — it should pick up the changed variable value

### Rollback

If a variable change breaks something, you have two options:
- **Git revert** — revert the commit and create a new release
- **Octopus UI** — edit the variable in the UI (which creates a new Git commit)

---

