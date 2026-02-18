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

Include the "Common Config" library variable set via **Variables → Library Sets**.

### Production Approval Gate

Add a **Manual Intervention** step to the deployment process and scope it to the **Production** environment only (under Conditions → Environments). Place it BEFORE the Helm step. Dev and staging deploys skip it automatically.

---

## Chapter 2: fraud-detector & Runbooks

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

### Cluster Health Check Runbook

Create a `cluster-ops` project in the **Platform** space. Add a "Health Check" runbook with a **Run a Script** step, target tag `platform-k8s`:

```bash
echo "============================================"
echo "  CLUSTER HEALTH CHECK"
echo "  Environment: #{Octopus.Environment.Name}"
echo "  Checked by:  #{Octopus.Deployment.CreatedBy.DisplayName}"
echo "  Time:        $(date -u '+%Y-%m-%d %H:%M:%S UTC')"
echo "============================================"

echo ""
echo "--- Nodes ---"
kubectl get nodes -o wide

echo ""
echo "--- Pods Not Running ---"
NOT_RUNNING=$(kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded --no-headers 2>/dev/null | wc -l)
if [ "$NOT_RUNNING" -gt 0 ]; then
  echo "⚠️  ${NOT_RUNNING} pods in bad state:"
  kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
else
  echo "✅ All pods healthy"
fi

echo ""
echo "--- Recent Warnings ---"
kubectl get events -A --field-selector type=Warning --sort-by='.lastTimestamp' 2>/dev/null | tail -10 || echo "No recent warnings"
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

### Compliance Gate

Manual Intervention step scoped to **Staging AND Production** (not just prod). Place it before the deploy step.

---

## Chapter 4: Tenants & Config-as-Code

### Tenant-Scoped Variables

Each tenant needs its own namespace, brand color, and data region — scoped by both tenant AND environment:

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

Use the package version variable to override Helm values. You can do this with additional `--set` arguments in the Helm step:

```
image.repository=<your-username>/finpay-payments-api
image.tag=#{Octopus.Action.Package[finpay-payments-api].PackageVersion}
```

Or use raw values YAML in the Helm step's values section.

### 4. Create Auto-Release Trigger

**Project → Triggers** → create a trigger that watches for new container image versions and auto-creates a release.

### Channels / Hotfix Path

Create a second channel called "Hotfix" and assign it the Hotfix lifecycle. In the channel's version rules, add a rule that routes image tags matching `*-hotfix` to this channel. All other tags flow through the default channel.

---

## Chapter 7: ArgoCD Integration

### ArgoCD Gateway

Go to **Infrastructure → Argo CD Instances → Add Argo CD Instance** in the Payments space.

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
echo "  Run by:      #{Octopus.Deployment.CreatedBy.DisplayName}"
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

**Step 3 — Verification script:**

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

