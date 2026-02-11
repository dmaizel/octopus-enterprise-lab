# Enterprise Lab: FinPay on Octopus Deploy

*A scenario-driven exercise simulating a real mid-size company adopting Octopus Deploy with an Enterprise Cloud license. Each chapter is a situation someone at FinPay actually faces.*

---

## The Company

**FinPay** — Mid-size fintech, 120 engineers, Series C, SOC 2 compliant. They process payments for online merchants across the US and Europe.

```
FinPay Engineering
│
├── Platform Engineering (8 engineers)
│   Owns: K8s clusters, CI/CD platform, observability, shared infrastructure
│   Lead: Sarah Chen
│
├── Payments Domain (25 engineers, 3 squads)
│   Services: payments-api, ledger-service, fraud-detector
│   Lead: Marcus Webb
│
└── Merchant Domain (20 engineers, 2 squads)
│   Services: merchant-portal, merchant-api, kyc-service
│   Lead: Priya Sharma
```

### Infrastructure

| Cluster | Purpose |
|---------|---------|
| `finpay-dev` | Development + Staging (namespace-isolated) |
| `finpay-prod` | Production |

### Octopus Spaces

| Space | Team | Why Separate |
|-------|------|-------------|
| `Platform` | Platform Engineering | Infra concerns shouldn't clutter product teams |
| `Payments` | Payments Domain | PCI-scoped services, strict deployment controls |
| `Merchants` | Merchant Domain | Customer-facing, tenanted deployments |

**Time estimate:** 3-4 hours across sessions.

---

## Setting the Stage

Before the scenarios begin, you need the local infrastructure and Octopus instance ready. This is the "Platform team's first day" — Sarah's team is setting up the deployment platform for the company.

### Prerequisites

```bash
brew install kind kubectl helm
# Verify
docker info > /dev/null 2>&1 && echo "✅ Docker running"
kind version && echo "✅ Kind"
kubectl version --client --short 2>/dev/null && echo "✅ kubectl"
helm version --short && echo "✅ Helm"
```

You also need:
- An **Octopus Cloud** trial (Enterprise features included): https://octopus.com/start
- A **GitHub** account

### Fork and Clone the Deploy Repo

The **[finpay-deploy](https://github.com/dmaizel/finpay-deploy)** repo is FinPay's deployment repository. It contains all Helm charts, Kubernetes manifests, and ArgoCD configs that FinPay's teams use. Fork it to your own GitHub account, then clone your fork:

```bash
git clone git@github.com:<YOUR_USERNAME>/finpay-deploy.git
```

> This repo is what a FinPay engineer would work with day-to-day. All services use `nginx:1.25-alpine` as the container image — the charts are structured like real production services (per-environment values, HPA, PDB, health checks), but the actual workload is just nginx.

### Run the Bootstrap

The bootstrap script (in *this* repo, not finpay-deploy) creates both Kind clusters, installs the NFS CSI driver (required by Octopus K8s Agents), and creates all the namespaces.

```bash
./setup/bootstrap.sh
```

### Verify

```bash
kubectl cluster-info --context kind-finpay-dev
kubectl cluster-info --context kind-finpay-prod
kubectl --context kind-finpay-dev get namespaces
kubectl --context kind-finpay-prod get namespaces
```

### Create the Octopus Spaces

Log into your Octopus Cloud instance (`https://<instance>.octopus.app`).

**Configuration → Spaces:**
1. Rename "Default" → `Platform`
2. Add Space → `Payments`
3. Add Space → `Merchants`

### Bootstrap Each Space

In **each of the 3 spaces** (switch via the top-left dropdown), create:

**Infrastructure → Environments:**
- `Development`
- `Staging`
- `Production`

**Library → Lifecycles → Add Lifecycle → "Standard":**

| Phase | Environment | Auto Deploy? |
|-------|-------------|-------------|
| 1 | Development | ✅ Yes |
| 2 | Staging | ❌ No |
| 3 | Production | ❌ No |

**Library → Lifecycles → Add Lifecycle → "Hotfix":**

| Phase | Environment | Auto Deploy? |
|-------|-------------|-------------|
| 1 | Staging | ❌ No |
| 2 | Production | ❌ No |

**Library → Git Credentials → Add:**
- Name: `GitHub`
- Username: your GitHub username
- Password: a Personal Access Token with `repo` scope

**Library → Variable Sets → Add → "Common Config":**

| Variable | Value | Scope |
|----------|-------|-------|
| `DockerRegistry` | `finpay.azurecr.io` | *(all)* |
| `KafkaBrokers` | `kafka.finpay.internal:9092` | Development |
| `KafkaBrokers` | `kafka.finpay.internal:9092` | Staging |
| `KafkaBrokers` | `kafka-1.finpay.internal:9092,kafka-2.finpay.internal:9092` | Production |

> You just created the same 3 environments, 2 lifecycles, 1 Git credential, and 1 variable set **three times** — once per space. Remember that feeling.

### Install K8s Agents

Each space needs its own agent per cluster. For each row in this table, go to the appropriate space, navigate to **Infrastructure → Deployment Targets → Add → Kubernetes Agent**, and run the wizard-generated Helm command against the right cluster.

| Space | Cluster Context | Agent Name | Target Tag | Environments |
|-------|----------------|------------|------------|-------------|
| Platform | `kind-finpay-dev` | `platform-dev` | `platform-k8s` | Development, Staging |
| Platform | `kind-finpay-prod` | `platform-prod` | `platform-k8s` | Production |
| Payments | `kind-finpay-dev` | `payments-dev` | `payments-k8s` | Development, Staging |
| Payments | `kind-finpay-prod` | `payments-prod` | `payments-k8s` | Production |
| Merchants | `kind-finpay-dev` | `merchants-dev` | `merchants-k8s` | Development, Staging |
| Merchants | `kind-finpay-prod` | `merchants-prod` | `merchants-k8s` | Production |

> **⚠️** The wizard JWT expires in 1 hour. If you take a break, regenerate.

After installing all 6 agents, take stock:

```bash
echo "=== Dev cluster: Octopus footprint ==="
kubectl --context kind-finpay-dev get pods -A | grep octopus
echo ""
echo "Namespaces: $(kubectl --context kind-finpay-dev get ns | grep -c octopus)"
echo "Pods: $(kubectl --context kind-finpay-dev get pods -A | grep -c octopus)"

echo ""
echo "=== Prod cluster: Octopus footprint ==="
kubectl --context kind-finpay-prod get pods -A | grep octopus
echo ""
echo "Namespaces: $(kubectl --context kind-finpay-prod get ns | grep -c octopus)"
echo "Pods: $(kubectl --context kind-finpay-prod get pods -A | grep -c octopus)"
```

Write down the total. This is the cost of the multi-space model before a single line of application code has been deployed.

---

*The stage is set. FinPay has clusters, an Octopus instance with 3 spaces, and agents installed. Now the real work begins.*

---

## Chapter 1: "We Need payments-api in Staging by Lunch"

**Monday morning.** Marcus Webb's team just merged a new refund flow into `payments-api`. QA wants it in staging for testing by lunch, and the risk team needs to sign off before it goes to production on Wednesday.

This is the bread-and-butter use case: get a Helm-based service deployed through environments with proper approvals.

### What You'll Do

Switch to the **Payments** space.

#### Step 1: Create the Project

**Projects → Add Project:**

| Field | Value |
|-------|-------|
| Name | `payments-api` |
| Lifecycle | Standard |

#### Step 2: Define Environment-Specific Config

Marcus's team uses different database endpoints and log levels per environment. In Octopus, this is done with scoped variables.

**Variables → Project Variables:**

| Variable | Value | Scope: Environment |
|----------|-------|--------------------|
| `Namespace` | `payments-dev` | Development |
| `Namespace` | `payments-staging` | Staging |
| `Namespace` | `payments-prod` | Production |

Also include shared company config:
**Variables → Library Sets → Include → "Common Config"**

#### Step 3: Build the Deployment Process

**Process → Add Step → Deploy a Helm Chart:**

| Setting | Value |
|---------|-------|
| Step name | `Deploy payments-api` |
| Target tags | `payments-k8s` |
| Chart source | Git Repository |
| Repository URL | `https://github.com/<YOUR_USERNAME>/finpay-deploy.git` |
| Git credential | `GitHub` |
| Branch | `main` |
| Chart path | `charts/payments-api` |
| Values file 1 | `charts/payments-api/values.yaml` |
| Values file 2 | `charts/payments-api/values-#{Octopus.Environment.Name \| ToLower}.yaml` |
| Helm release name | `payments-api` |
| Namespace | `#{Namespace}` |

#### Step 4: Add Production Approval

The risk team wants to sign off before anything hits production. In Octopus, this is a Manual Intervention step scoped to an environment.

**Process → Add Step → Manual Intervention Required:**

| Setting | Value |
|---------|-------|
| Step name | `Risk Team Approval` |
| Instructions | `Approve deployment of payments-api to Production. Verify staging tests passed and risk assessment is complete.` |
| Responsible teams | Everyone (or create a "Risk Team" in Configuration → Teams) |
| **Conditions → Environments** | Production only |

**Reorder** this step to be BEFORE the Helm step.

This means: Dev and Staging deploy immediately. Production pauses and waits for a human.

#### Step 5: Ship It

**Create Release → v1.0.0**

The Standard lifecycle auto-deploys to Development. Watch it:
```bash
kubectl --context kind-finpay-dev get pods -n payments-dev -w
```

Now promote to Staging — click **Deploy** next to Staging in the release view.
```bash
kubectl --context kind-finpay-dev get pods -n payments-staging
```

Marcus sends the staging URL to QA. It's 11:45. Made it before lunch.

#### Step 6: Wednesday Production Push

QA approves. Marcus promotes to Production. The deployment pauses at "Risk Team Approval" — someone needs to click approve in Octopus.

Approve it. Watch the production deploy:
```bash
kubectl --context kind-finpay-prod get pods -n payments-prod
```

### 📝 What to Notice

- How many clicks from "code ready" to "running in staging"?
- Check the deployment log — can you see what Helm commands Octopus actually ran?
- The Manual Intervention step only fires for Production. Dev and Staging skip it entirely. Is this obvious from the process view?
- The release `v1.0.0` is an immutable snapshot. The same artifact flows through environments. Compare this to GitOps where each environment tracks a Git ref independently.

---

## Chapter 2: "Fraud-detector Is Acting Up in Staging"

**Tuesday afternoon.** A Slack message from the fraud team: *"fraud-detector is returning false positives on every transaction in staging. Can someone restart it?"*

Junior developer Alex doesn't have `kubectl` access to the staging cluster — FinPay restricts direct cluster access to the Platform team. But Alex should be able to restart a service in staging through Octopus without needing cluster credentials.

### What You'll Do

This chapter covers two things: deploying a second service (fraud-detector) and creating operational runbooks.

#### Step 1: Deploy fraud-detector

Still in the **Payments** space.

**Projects → Add Project:**

| Field | Value |
|-------|-------|
| Name | `fraud-detector` |
| Lifecycle | Standard |

**Variables → Project Variables:**

| Variable | Value | Scope |
|----------|-------|-------|
| `Namespace` | `fraud-dev` | Development |
| `Namespace` | `fraud-staging` | Staging |
| `Namespace` | `fraud-prod` | Production |

**Process → Add Step → Deploy a Helm Chart:**

Same pattern as payments-api:
- Target tags: `payments-k8s`
- Chart path: `charts/fraud-detector`
- Values: `charts/fraud-detector/values.yaml` + `charts/fraud-detector/values-#{Octopus.Environment.Name | ToLower}.yaml`
- Release name: `fraud-detector`
- Namespace: `#{Namespace}`

**Create Release → v1.0.0 → Deploy through to Staging.**

Verify it's running:
```bash
kubectl --context kind-finpay-dev get pods -n fraud-staging
```

#### Step 2: Create the "Restart Service" Runbook

This is what Alex will use instead of `kubectl`.

**fraud-detector → Operations → Runbooks → Add Runbook:**

| Field | Value |
|-------|-------|
| Name | `Restart Service` |

**Process → Add Step → Run a Script:**

| Setting | Value |
|---------|-------|
| Step name | `Restart` |
| Target tags | `payments-k8s` |

```bash
echo "Restarting fraud-detector in #{Octopus.Environment.Name}..."
kubectl rollout restart deployment/fraud-detector -n #{Namespace}
kubectl rollout status deployment/fraud-detector -n #{Namespace} --timeout=120s
echo ""
echo "Pod status after restart:"
kubectl get pods -n #{Namespace} -l app=fraud-detector
```

#### Step 3: Be Alex

Run the runbook. Pick environment: **Staging**.

The runbook executes on the K8s Agent (which has cluster access), but Alex never touches `kubectl` directly. The Octopus audit log records who ran it and when.

```bash
# Verify the restart happened (new pod age)
kubectl --context kind-finpay-dev get pods -n fraud-staging -l app=fraud-detector
```

#### Step 4: Create a "Cluster Health Check" Runbook

Sarah (Platform lead) wants a quick way to check cluster health without SSHing in. Switch to the **Platform** space.

**Projects → Add Project → `cluster-ops`** (Lifecycle: Standard)

**cluster-ops → Runbooks → Add Runbook → "Health Check":**

**Process → Add Step → Run a Script:**

| Setting | Value |
|---------|-------|
| Step name | `Cluster Health Check` |
| Target tags | `platform-k8s` |

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
echo "--- Pods Not Running (across all namespaces) ---"
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

echo ""
echo "--- PVCs ---"
kubectl get pvc -A
```

Run it against Development. Then against Production.

### 📝 What to Notice

- The runbook requires you to pick an environment, even though "cluster health" isn't environment-specific. The dev cluster hosts both Development and Staging — picking either one runs on the same cluster.
- Alex never got `kubectl` access, but could restart the service. The audit trail shows exactly who did what and when. This is genuinely useful for compliance.
- The Platform space runbook can't see or affect anything in the Payments space. Sarah's health check runs on the Platform agent, which *happens* to be on the same cluster as the Payments agent, but they're completely independent in Octopus.

---

## Chapter 3: "KYC-Service Needs to Launch — and It Handles PII"

**Wednesday.** Priya's Merchant team has built `kyc-service` — it processes identity documents for merchant onboarding. It handles PII (personally identifiable information), so compliance requires:

1. Every production deploy must be explicitly approved
2. The service runs with a `compliance: pii-handler` label
3. Deployment config must be auditable

KYC-service doesn't use Helm — the team prefers raw Kubernetes YAML. This is common in real companies: not every team standardizes on the same tooling.

### What You'll Do

Switch to the **Merchants** space.

#### Step 1: Create the Project

**Projects → Add Project:**

| Field | Value |
|-------|-------|
| Name | `kyc-service` |
| Lifecycle | Standard |

#### Step 2: Variables

The raw YAML in `manifests/kyc-service/deployment.yaml` uses Octopus variable substitution (`#{VariableName}`). These get replaced at deploy time.

**Variables → Project Variables:**

| Variable | Value | Scope |
|----------|-------|-------|
| `Namespace` | `kyc-dev` | Development |
| `Namespace` | `kyc-staging` | Staging |
| `Namespace` | `kyc-prod` | Production |
| `Replicas` | `1` | Development |
| `Replicas` | `2` | Staging |
| `Replicas` | `2` | Production |
| `LogLevel` | `debug` | Development |
| `LogLevel` | `info` | Staging |
| `LogLevel` | `warn` | Production |
| `DocumentStorageUrl` | `s3://finpay-dev-kyc-docs` | Development |
| `DocumentStorageUrl` | `s3://finpay-staging-kyc-docs` | Staging |
| `DocumentStorageUrl` | `s3://finpay-prod-kyc-docs` | Production |

#### Step 3: Compliance Approval

Because this service handles PII, compliance requires explicit approval for ALL environments — not just production.

**Process → Add Step → Manual Intervention Required:**

| Setting | Value |
|---------|-------|
| Step name | `Compliance Sign-Off` |
| Instructions | `kyc-service handles PII. Confirm data handling review is complete for #{Octopus.Environment.Name}.` |
| Responsible teams | Everyone |
| **Conditions → Environments** | Staging, Production |

#### Step 4: Deploy Raw YAML

**Process → Add Step → Deploy Kubernetes YAML:**

| Setting | Value |
|---------|-------|
| Step name | `Deploy kyc-service` |
| Target tags | `merchants-k8s` |
| YAML source | Git Repository |
| Repository URL | `https://github.com/<YOUR_USERNAME>/finpay-deploy.git` |
| Git credential | `GitHub` |
| Branch | `main` |
| Paths | `manifests/kyc-service/deployment.yaml` |

#### Step 5: Deploy and Observe

**Create Release → v1.0.0**

Auto-deploys to Development (no approval needed for dev). Then promote to Staging — the Compliance Sign-Off step fires. Approve it.

```bash
kubectl --context kind-finpay-dev get pods -n kyc-dev
kubectl --context kind-finpay-dev get pods -n kyc-staging
```

### 📝 What to Notice

- Open `manifests/kyc-service/deployment.yaml` in Git. It says `replicas: #{Replicas}`. Now check what's actually in the cluster: `kubectl --context kind-finpay-dev get deployment kyc-service -n kyc-staging -o yaml | grep replicas`. The variable got substituted. **What's in Git is NOT what's running.** This is the fundamental difference between Octopus variable substitution and GitOps.
- The Manual Intervention step scoped to Staging + Production means dev deploys fly through, but staging and prod pause for approval. Different compliance posture per environment, same pipeline.
- Compare the project setup effort: raw YAML required more variables (you had to define `Replicas` and `LogLevel` per environment). With Helm, these would live in values files. The tradeoff: raw YAML is simpler to understand but pushes more config into Octopus variables.

---

## Chapter 4: "EuroShop Just Signed — Spin Up Their Environment"

**Thursday.** The sales team closes EuroShop, a European merchant. FinPay hosts the merchant portal per-customer, each with isolated config (branding, data region, API keys). The existing customer is Acme Corp.

Priya's team needs to deploy `merchant-portal` for EuroShop alongside Acme Corp's existing deployment, using the same codebase but different configuration.

This is what Octopus **Tenants** are for.

### What You'll Do

Still in the **Merchants** space.

#### Step 1: Create the Tenants

**Infrastructure → Tenants → Add Tenant:**

| Tenant | Description |
|--------|-------------|
| `Acme Corp` | US-based enterprise merchant. Existing customer. |
| `EuroShop` | European merchant. GDPR requirements, EU data region. |

#### Step 2: Create merchant-portal Project

**Projects → Add Project:**

| Field | Value |
|-------|-------|
| Name | `merchant-portal` |
| Lifecycle | Standard |

**Settings → Multi-tenancy:**
- Change to **"Allow deployments with or without a tenant"**

#### Step 3: Connect Tenants to the Project

For each tenant:
1. **Infrastructure → Tenants → [tenant name]**
2. **Connect Project** → `merchant-portal`
3. Select environments: Development, Staging, Production

#### Step 4: Tenant-Scoped Variables

This is where tenants get interesting. The same variable has different values depending on which tenant you're deploying.

**merchant-portal → Variables → Project Variables:**

Base (untenanted):
| Variable | Value | Scope |
|----------|-------|-------|
| `Namespace` | `merchant-dev` | Development |
| `Namespace` | `merchant-staging` | Staging |
| `Namespace` | `merchant-prod` | Production |

Acme Corp:
| Variable | Value | Environment | Tenant |
|----------|-------|-------------|--------|
| `Namespace` | `merchant-acme-dev` | Development | Acme Corp |
| `Namespace` | `merchant-acme-staging` | Staging | Acme Corp |
| `Namespace` | `merchant-acme-prod` | Production | Acme Corp |
| `BrandColor` | `#FF6600` | *(all)* | Acme Corp |
| `DataRegion` | `us-east-1` | *(all)* | Acme Corp |

EuroShop:
| Variable | Value | Environment | Tenant |
|----------|-------|-------------|--------|
| `Namespace` | `merchant-euro-dev` | Development | EuroShop |
| `Namespace` | `merchant-euro-staging` | Staging | EuroShop |
| `Namespace` | `merchant-euro-prod` | Production | EuroShop |
| `BrandColor` | `#003399` | *(all)* | EuroShop |
| `DataRegion` | `eu-west-1` | *(all)* | EuroShop |

#### Step 5: Deployment Process

**Process → Add Step → Deploy a Helm Chart:**

| Setting | Value |
|---------|-------|
| Step name | `Deploy merchant-portal` |
| Target tags | `merchants-k8s` |
| Chart source | Git Repository |
| Repository URL | `https://github.com/<YOUR_USERNAME>/finpay-deploy.git` |
| Git credential | `GitHub` |
| Branch | `main` |
| Chart path | `charts/merchant-portal` |
| Values file 1 | `charts/merchant-portal/values.yaml` |
| Values file 2 | `charts/merchant-portal/values-#{Octopus.Environment.Name \| ToLower}.yaml` |
| Helm release name | `merchant-portal-#{Octopus.Deployment.Tenant.Name \| ToLower \| Replace " " "-"}` |
| Namespace | `#{Namespace}` |

> Note the release name includes the tenant name. Without this, both tenants would collide on the same Helm release.

#### Step 6: Deploy Both Tenants

**Create Release → v1.0.0**

Deploy to Development + **Acme Corp**:
```bash
kubectl --context kind-finpay-dev get pods -n merchant-acme-dev
```

Deploy to Development + **EuroShop**:
```bash
kubectl --context kind-finpay-dev get pods -n merchant-euro-dev
```

Same code, different namespaces, different config. From one project.

#### Step 7: The Tenant Dashboard

Go back to **Projects → merchant-portal → Overview**.

You should see a matrix: environments across the top, tenants down the side. Each cell shows the deployed version. This is the tenant dashboard — at a glance, you can see "Acme is on v1.0.0 in staging, EuroShop is still only in dev."

### 📝 What to Notice

- The variable matrix grows combinatorially: 2 tenants × 3 environments × 3 variables = 18 entries. With 50 merchants, that's 450+ variable entries. How does that scale?
- Compare to the K8s-native approach: you'd have `values-acme.yaml` and `values-euroshop.yaml` per environment. No Octopus tenant abstraction needed — just namespaces + values files. But you'd lose the dashboard.
- The tenant dashboard is the real win here. "Which merchants are on which version in which environment" — that's a question that's genuinely hard to answer with kubectl and ArgoCD.
- Adding a new merchant (tenant) requires: create the tenant, connect it to the project, add all the scoped variables, create the namespaces. For a self-service model, you'd want this automated via API.

---

## Chapter 5: "Friday Release Train"

**Friday morning.** Three things need to go to production today:

1. `payments-api` v1.1.0 — the refund flow, tested and approved
2. `fraud-detector` v1.1.0 — updated model with fixed false-positive threshold
3. `merchant-portal` v1.1.0 — for Acme Corp only (EuroShop isn't ready)

Marcus wants them deployed in order: payments-api first (it's the dependency), then fraud-detector, then merchant-portal. Each needs production approval.

### What You'll Do

#### Step 1: Add a Post-Deploy Smoke Test to payments-api

Before promoting fraud-detector, Marcus wants proof that payments-api is healthy.

**Payments space → payments-api → Process:**

**Add Step → Run a Script** (after the Helm step):

| Setting | Value |
|---------|-------|
| Step name | `Smoke Test` |
| Target tags | `payments-k8s` |

```bash
echo "Running smoke test for payments-api in #{Octopus.Environment.Name}..."

NAMESPACE="#{Namespace}"
kubectl rollout status deployment/payments-api -n ${NAMESPACE} --timeout=120s

POD=$(kubectl get pod -n ${NAMESPACE} -l app=payments-api -o jsonpath='{.items[0].metadata.name}')
STATUS=$(kubectl exec -n ${NAMESPACE} ${POD} -- curl -s -o /dev/null -w '%{http_code}' http://localhost:80/ 2>/dev/null)

if [ "${STATUS}" = "200" ]; then
  echo "✅ payments-api healthy (HTTP ${STATUS})"
else
  echo "❌ payments-api unhealthy (HTTP ${STATUS})"
  exit 1
fi
```

#### Step 2: Release and Promote

**payments-api:** Create Release v1.1.0 → Deploy to Development → Promote to Staging → Promote to Production (approve the Risk Team intervention).

Watch the smoke test run after the Helm deploy. If it fails, the deployment is marked failed and the release doesn't "pass" production.

**fraud-detector:** Create Release v1.1.0 → Deploy through to Production.

**merchant-portal:** Create Release v1.1.0 → Deploy to Production + **Acme Corp only** (skip EuroShop).

```bash
# Verify everything is running in production
kubectl --context kind-finpay-prod get pods -n payments-prod
kubectl --context kind-finpay-prod get pods -n fraud-prod
kubectl --context kind-finpay-prod get pods -n merchant-acme-prod
```

### 📝 What to Notice

- There's no built-in way to say "deploy these 3 projects in sequence as a single release train." Each project has its own release and its own promotion. Coordination is manual (or via API automation).
- The smoke test step runs on the same K8s Agent as the deploy. It can `kubectl exec` into the pod. This is powerful — you're running integration tests as part of the deployment pipeline, with access to the actual cluster.
- Tenanted deploy (merchant-portal for Acme only) means EuroShop is untouched. The tenant model isolates releases per customer. Compare to non-tenanted: you'd need separate projects or complex variable logic.
- Check the **audit trail**: **Projects → payments-api → Releases → v1.1.0**. You can see every deployment, every approval, every environment, timestamped with who did what. This is the compliance story.

---

## Chapter 6: "The Compliance Audit"

**The following Monday.** FinPay's SOC 2 auditor shows up (remotely). They want answers to specific questions about production deployment controls. This isn't hypothetical — every fintech deals with this.

### The Auditor's Questions

Work through these using the Octopus UI. The point isn't to automate them — it's to see what information is readily available and what's hard to find.

#### Question 1: "Show me every production deployment in the last 30 days."

**Where to look:** Each space's **Deployments** tab (Infrastructure → Deployments, or the main dashboard). Filter by environment: Production.

Can you get a cross-space view (all production deployments across Platform + Payments + Merchants in one screen)?

| Can you answer it? | Where? | Easy or Hard? |
|---------------------|--------|--------------|
| | | |

#### Question 2: "Who approved each production deployment?"

**Where to look:** Click into any production deployment → look for the Manual Intervention step → see who approved and when.

Is the approver clearly visible? Is it in the deployment summary or do you need to drill into the step?

| Can you answer it? | Where? | Easy or Hard? |
|---------------------|--------|--------------|
| | | |

#### Question 3: "What changed between the staging deploy and the production deploy of payments-api?"

**Where to look:** Release → Deployment to Staging vs Deployment to Production. Same release = same artifacts. But were the variables different? Was the Helm values override different?

Can you diff what was actually applied to staging vs production?

| Can you answer it? | Where? | Easy or Hard? |
|---------------------|--------|--------------|
| | | |

#### Question 4: "Show me that kyc-service (PII handler) cannot be deployed to production without approval."

**Where to look:** kyc-service → Process → the Manual Intervention step scoped to Staging + Production.

Is this policy *enforced* (can't be bypassed) or just *configured* (an admin could remove the step)?

| Can you answer it? | Where? | Easy or Hard? |
|---------------------|--------|--------------|
| | | |

#### Question 5: "Can a developer in the Payments team deploy to the Merchant team's production environment?"

**Where to look:** Configuration → Teams and Roles (in each space). Space-level RBAC controls who can deploy where.

Is space isolation sufficient? Remember the K8s agents are on the same cluster — does Octopus RBAC prevent cross-namespace deploys, or is that a K8s RBAC concern?

| Can you answer it? | Where? | Easy or Hard? |
|---------------------|--------|--------------|
| | | |

#### Question 6: "Show me a history of all operational actions (restarts, health checks) run against production."

**Where to look:** The Runbooks section of each project. Runbook runs have their own audit log.

Is there a unified view of "all runbook runs against production" across projects?

| Can you answer it? | Where? | Easy or Hard? |
|---------------------|--------|--------------|
| | | |

### 📝 What to Notice

- Some of these questions are easy to answer per-space, hard to answer cross-space. There's no "all production deployments across the entire Octopus instance" view.
- The audit trail per deployment is solid — who, what, when, which release, which approval. This is a genuine Octopus strength.
- The gap between Octopus RBAC (who can deploy a project) and K8s RBAC (what the agent can do in-cluster) is a real compliance concern. A misconfigured agent could deploy to any namespace, even if the Octopus project variables are set correctly.

---

## Chapter 7: "Sarah Wants to Try the ArgoCD Path"

**Two weeks later.** Sarah (Platform lead) has been watching the Payments team's Octopus experience. The native Helm deployment works, but she's bothered by a few things:

1. Variable substitution means Git isn't the source of truth
2. No drift detection — if someone `kubectl edit`s in production, nobody knows
3. The K8s Agent is heavy (3+ pods per space per cluster)

She's heard about Octopus's ArgoCD integration and wants to try it. The idea: Octopus handles promotion and approval, ArgoCD handles the actual sync. Best of both worlds?

### What You'll Do

#### Step 1: Install ArgoCD on the Dev Cluster

```bash
kubectl config use-context kind-finpay-dev

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server \
  -n argocd --timeout=180s

ARGO_PASS=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD admin password: ${ARGO_PASS}"

# Port-forward for UI access
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
echo "ArgoCD UI: https://localhost:8080"
```

#### Step 2: Generate an ArgoCD Token

```bash
brew install argocd  # if not installed
argocd login localhost:8080 --insecure --username admin --password "${ARGO_PASS}"
ARGO_TOKEN=$(argocd account generate-token --account admin)
echo "Token: ${ARGO_TOKEN}"
```

#### Step 3: Install the Octopus ArgoCD Gateway

In the **Payments** space: **Infrastructure → Argo CD Instances → Add Argo CD Instance**

The wizard generates a Helm command. Run it against the dev cluster:

```bash
kubectl config use-context kind-finpay-dev

helm upgrade --install --atomic \
  --create-namespace --namespace octo-argo-gateway \
  --version "*.*" \
  --set registration.octopus.name="finpay-argocd-dev" \
  --set registration.octopus.serverApiUrl="https://<instance>.octopus.app/" \
  --set registration.octopus.serverAccessToken="<JWT_FROM_WIZARD>" \
  --set registration.octopus.environments="{development,staging}" \
  --set registration.octopus.spaceId="<PAYMENTS_SPACE_ID>" \
  --set gateway.octopus.serverGrpcUrl="grpc://<instance>.octopus.app:8443" \
  --set gateway.argocd.serverGrpcUrl="grpc://argocd-server.argocd.svc.cluster.local" \
  --set gateway.argocd.insecure="true" \
  --set gateway.argocd.plaintext="false" \
  --set gateway.argocd.authenticationToken="${ARGO_TOKEN}" \
  finpay-argocd-dev \
  oci://registry-1.docker.io/octopusdeploy/octopus-argocd-gateway-chart
```

Compare the footprint:
```bash
echo "=== ArgoCD Gateway pods ==="
kubectl --context kind-finpay-dev get pods -n octo-argo-gateway

echo ""
echo "=== K8s Agent pods (for comparison) ==="
kubectl --context kind-finpay-dev get pods -A | grep octopus-agent
```

The Gateway is 1 pod. The K8s Agent is 3+.

#### Step 4: Create ArgoCD Applications

These are standard ArgoCD Applications, with Octopus annotations that map them to projects and environments:

```bash
# Replace <YOUR_USERNAME> with your GitHub username
cat > /tmp/argocd-apps.yaml << 'EOF'
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-api-dev
  namespace: argocd
  annotations:
    argo.octopus.com/project: payments-api-argo
    argo.octopus.com/environment: development
spec:
  project: default
  source:
    repoURL: https://github.com/<YOUR_USERNAME>/finpay-deploy.git
    targetRevision: main
    path: argocd-manifests/payments-api/development
  destination:
    server: https://kubernetes.default.svc
    namespace: payments-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-api-staging
  namespace: argocd
  annotations:
    argo.octopus.com/project: payments-api-argo
    argo.octopus.com/environment: staging
spec:
  project: default
  source:
    repoURL: https://github.com/<YOUR_USERNAME>/finpay-deploy.git
    targetRevision: main
    path: argocd-manifests/payments-api/staging
  destination:
    server: https://kubernetes.default.svc
    namespace: payments-staging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

kubectl --context kind-finpay-dev apply -f /tmp/argocd-apps.yaml
```

Check ArgoCD picked them up:
```bash
argocd app list
```

#### Step 5: Create the Octopus Project (ArgoCD Mode)

In the **Payments** space:

**Projects → Add Project:**

| Field | Value |
|-------|-------|
| Name | `payments-api-argo` |
| Lifecycle | Standard |

**Process → Add Step → Update Argo CD Application Image Tags:**

| Setting | Value |
|---------|-------|
| Step name | `Update Image Tag` |
| Container image | `nginx` |

#### Step 6: Deploy and Compare

**Create Release → v1.0.0 → Deploy to Development**

What happens:
1. Octopus finds ArgoCD Applications annotated with `project: payments-api-argo` + `environment: development`
2. Updates the image tag in the Git repo
3. ArgoCD detects the Git change and syncs

```bash
kubectl --context kind-finpay-dev get pods -n payments-dev
```

#### Step 7: Test Drift Detection (The Real Win)

This is what Sarah was after. Simulate someone making a manual change in the cluster:

```bash
kubectl --context kind-finpay-dev set image deployment/payments-api \
  payments-api=nginx:1.24-alpine -n payments-dev
```

Now watch:
```bash
# ArgoCD should revert it within seconds
kubectl --context kind-finpay-dev get pods -n payments-dev -w
```

ArgoCD's self-heal reverts the manual change. **This is something native Octopus deployment can't do.** It deploys and walks away — if someone edits in-cluster, Octopus has no idea.

### 📝 What to Notice

- **Gateway vs Agent footprint:** 1 pod vs 3+. The Gateway is purpose-built and much lighter.
- **The model shift:** Octopus goes from "I run helm upgrade" to "I commit to Git." The actual deployment is ArgoCD's job. Octopus adds promotion gates and audit trail on top.
- **Annotations as glue:** The `argo.octopus.com/*` annotations on ArgoCD Applications are how Octopus knows which app to update. Config lives on the Kubernetes resource, not in the Octopus UI. That's more GitOps-native.
- **The drift detection test:** Did ArgoCD revert the change? How fast? This is the reconciliation loop that native Octopus lacks.
- **The open question:** The Gateway registered to the Payments space. Could the Merchants space use the same Gateway and ArgoCD instance? Or would they need their own?

---

## Chapter 8: "The Database Migration"

**Wednesday of week 3.** The Payments team needs to run a database migration for payments-api before deploying v1.2.0. Migrations are risky — they need to be trackable, approvable, and never accidentally run twice.

### What You'll Do

Switch to the **Payments** space.

#### Step 1: Create a Migration Runbook

**payments-api → Operations → Runbooks → Add Runbook:**

| Field | Value |
|-------|-------|
| Name | `Run DB Migration` |

**Process:**

**Step 1: Manual Intervention (scoped to Staging + Production):**

| Setting | Value |
|---------|-------|
| Step name | `Approve Migration` |
| Instructions | `Database migration: #{MigrationName}. Confirm this has been tested in the previous environment.` |
| Conditions → Environments | Staging, Production |

**Step 2: Run a Script:**

| Setting | Value |
|---------|-------|
| Step name | `Execute Migration` |
| Target tags | `payments-k8s` |

```bash
echo "============================================"
echo "  DATABASE MIGRATION"
echo "  Service:     payments-api"
echo "  Environment: #{Octopus.Environment.Name}"
echo "  Migration:   #{MigrationName}"
echo "  Run by:      #{Octopus.Deployment.CreatedBy.DisplayName}"
echo "  Time:        $(date -u '+%Y-%m-%d %H:%M:%S UTC')"
echo "============================================"
echo ""

# In production this would be something like:
# kubectl run migration-#{MigrationName} \
#   --image=finpay/payments-api:#{Octopus.Release.Number} \
#   --restart=Never \
#   -n #{Namespace} \
#   -- ./migrate.sh #{MigrationName}

echo "(Simulated) Running migration #{MigrationName}..."
sleep 2
echo "Migration complete. Rows affected: 42,387. Duration: 12.3s."
```

**Step 3: Run a Script (verification):**

```bash
echo "Verifying migration..."
echo "Checking payments-api health post-migration..."

POD=$(kubectl get pod -n #{Namespace} -l app=payments-api -o jsonpath='{.items[0].metadata.name}')
STATUS=$(kubectl exec -n #{Namespace} ${POD} -- curl -s -o /dev/null -w '%{http_code}' http://localhost:80/ 2>/dev/null)

if [ "${STATUS}" = "200" ]; then
  echo "✅ payments-api healthy after migration"
else
  echo "⚠️  payments-api returned HTTP ${STATUS} — investigate"
fi
```

**Add a Prompted Variable** (set when running the runbook):

Go to **Variables → Project Variables** and add:
| Variable | Label | Prompt |
|----------|-------|--------|
| `MigrationName` | Migration name | ✅ Prompt (e.g., "V1.2__add_refund_column") |

Mark it as a prompted variable (check "Prompt for value during deployment").

#### Step 2: Run the Migration

Run the runbook against **Development** first. Enter `V1.2__add_refund_column` as the migration name.

Then run against **Staging** — notice the approval step fires.

Check the run history: **Runbooks → Run DB Migration → runs**. Each run shows the migration name, who ran it, which environment, and when.

### 📝 What to Notice

- Prompted variables let the operator provide context at runtime. The migration name is recorded in the audit trail — you can always trace back which migration ran where.
- The approval for staging + production means you can't accidentally run a migration against prod without someone signing off.
- Runbook runs have their own history, separate from deployments. You can see "migration X ran against staging at 14:22 by Marcus, approved by Sarah."

---

## The Retrospective

You've lived through 3 weeks at FinPay. Step back and assess.

### Footprint Check

```bash
echo "=== TOTAL OCTOPUS FOOTPRINT ==="
echo ""
echo "Dev cluster:"
echo "  Pods: $(kubectl --context kind-finpay-dev get pods -A | grep -c octopus)"
echo "  Namespaces: $(kubectl --context kind-finpay-dev get ns | grep -c octopus)"
echo "  PVCs: $(kubectl --context kind-finpay-dev get pvc -A | grep -c octopus)"
echo ""
echo "Prod cluster:"
echo "  Pods: $(kubectl --context kind-finpay-prod get pods -A | grep -c octopus)"
echo "  Namespaces: $(kubectl --context kind-finpay-prod get ns | grep -c octopus)"
echo "  PVCs: $(kubectl --context kind-finpay-prod get pvc -A | grep -c octopus)"
```

### The Duplication Tally

| What | How Many Times? |
|------|----------------|
| "Development" environment created | |
| "Standard" lifecycle created | |
| "Common Config" variable set created | |
| GitHub Git credential stored | |
| K8s Agent installations (total) | |
| If Kafka broker address changes, places to update | |

### Where Octopus Earned Its Keep

| Feature | The Scenario That Proved It |
|---------|-----------------------------|
| Approval gates | |
| Audit trail | |
| Runbooks with RBAC | |
| Tenant dashboard | |
| Lifecycle enforcement | |
| Deployment visibility | |

### Where Octopus Created Friction

| Pain Point | The Scenario Where You Felt It |
|------------|-------------------------------|
| Per-space agent duplication | |
| No cross-space visibility | |
| Environment forced on cluster-level ops | |
| Variable substitution breaks Git-as-truth | |
| No release train coordination | |
| Variable matrix explosion with tenants | |
| Duplicate config across spaces | |

### Concept Map

| Octopus Concept | What It Replaced | Better or Worse? | Notes |
|-----------------|------------------|-------------------|-------|
| Space | | | |
| Environment | | | |
| Lifecycle | | | |
| Release | | | |
| Target (K8s Agent) | | | |
| Worker | | | |
| Tenant | | | |
| Manual Intervention | | | |
| Runbook | | | |
| Variable substitution | | | |
| ArgoCD Gateway | | | |

---

## Cleanup

```bash
kind delete cluster --name finpay-dev
kind delete cluster --name finpay-prod
rm -rf ~/finpay-deploy
```

---

*Last updated: February 2026*
