# Enterprise Lab: FinPay on Octopus Deploy

*A scenario-driven exercise simulating a real mid-size company adopting Octopus Deploy with an Enterprise Cloud license. Each chapter is a situation someone at FinPay actually faces.*

> **📝 Before you start:** Open a Google Doc on the side. As you work through this exercise, document any friction points, unclear UX, confusing concepts, or any insights and comments that come to mind. Your observations are the most valuable output of this lab.

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
    Services: merchant-portal, merchant-api, kyc-service
    Lead: Priya Sharma
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

### People & Permissions

| Person | Role | Access |
|--------|------|--------|
| Sarah Chen | Platform Lead | Admin across all spaces |
| Marcus Webb | Payments Lead | Full access to Payments space, read-only elsewhere |
| Priya Sharma | Merchants Lead | Full access to Merchants space, read-only elsewhere |
| Alex Rivera | Junior Dev (Payments) | Can deploy to Dev/Staging in Payments, can run runbooks, cannot deploy to Production |

**Time estimate:** 3-4 hours across sessions.

> **💡 Stuck?** There's an [Answer Key](ANSWER_KEY.md) with step-by-step details for each section. Try to figure things out on your own first — the friction is where the learning happens. Use the answer key as a fallback, not a walkthrough.

---

## Setting the Stage

Before the scenarios begin, you need the local infrastructure and Octopus instance ready. This is the "Platform team's first day" — Sarah's team is setting up the deployment platform for the company.

### Prerequisites

```bash
brew install kind kubectl helm
```

You also need:
- **Docker Desktop** running
- An **Octopus Cloud** trial (Enterprise features included): https://octopus.com/start
- A **GitHub** account
- A **Docker Hub** account (free tier is fine): https://hub.docker.com/signup — needed for Chapter 5

### Fork and Clone the Deploy Repo

The **[finpay-deploy](https://github.com/dmaizel/finpay-deploy)** repo is FinPay's deployment repository — Helm charts, Kubernetes manifests, and ArgoCD configs. Fork it to your own GitHub account, then clone your fork:

```bash
git clone git@github.com:<YOUR_USERNAME>/finpay-deploy.git
```

> All services use `nginx:1.25-alpine` as the container image. The charts are structured like real production services (per-environment values, HPA, PDB, health checks), but the actual workload is just nginx.

### Run the Bootstrap

The bootstrap script (in *this* repo, not finpay-deploy) creates both Kind clusters, installs prerequisites, and creates all the namespaces.

```bash
./setup/bootstrap.sh
```

### Create the Octopus Spaces

Log into your Octopus Cloud instance.

1. Rename "Default" → `Platform`
2. Create `Payments`
3. Create `Merchants`

### Bootstrap Each Space

In **each of the 3 spaces**, create:

- **Environments:** Development, Staging, Production
- **Lifecycles:**
  - "Standard" — Development (auto-deploy) → Staging (manual) → Production (manual)
  - "Hotfix" — Staging (manual) → Production (manual)
- **Git Credentials** — so Octopus can pull from your finpay-deploy fork
- **Library Variable Set "Common Config"** — shared values like registry URL and Kafka broker addresses (use environment scoping for values that differ between dev/staging/prod)

> You just set up the same environments, lifecycles, credentials, and variable sets **three times** — once per space.

### Set Up External Feeds

FinPay's CI pipeline pushes container images to Docker Hub. Set up a Docker Container Registry feed in both the **Payments** and **Merchants** spaces so Octopus can detect new images and trigger releases.

You'll need to find where external feeds are configured and add Docker Hub as a container registry. No credentials are needed for public images at this stage.

### Set Up RBAC

FinPay has different access levels per team. In each space, configure teams and roles to match the org structure described in "People & Permissions" above.

**Configuration → Teams** — Create teams and assign users with appropriate roles:

- Sarah's Platform team needs admin access across all spaces
- Marcus's developers can deploy to Dev and Staging in Payments, but Production requires a specific role
- Alex (junior dev) should be able to trigger deployments to Dev/Staging and run runbooks, but not deploy to Production or modify project configuration
- Priya's team should have full control of the Merchants space but only read access to Payments

Explore the built-in roles and figure out the right combination. Consider: what's the difference between a "Deployment Creator" and a "Project Deployer"? Where do you control "can run runbooks but can't deploy to production"?

**Think about:** Alex can't deploy to production, but what if Alex's code is bundled in the same release as Marcus's? Does restricting Alex from deploying actually protect anything? Where should the safety net live — RBAC, PR approvals, or the CI/CD pipeline itself?

### Verify RBAC (Recommended)

Configuring roles is one thing — proving they work is another. Actually log in as the users you created and verify the permissions behave as expected.

**The challenge:** Octopus Cloud uses Octopus ID (email-based auth), so each test user needs a real email address. The simplest approach is **Gmail aliases** — Gmail treats `yourname+anything@gmail.com` as the same inbox:

1. Invite test users via **Configuration → Users → Invite User**:
   - `yourname+sarah@gmail.com` (Sarah Chen — Platform Admin)
   - `yourname+alex@gmail.com` (Alex Rivera — Junior Dev, Payments)
2. Accept the invite emails from your inbox (they all arrive at your real address)
3. Open an **incognito/private window** and log in as each user

**What to test:**
- As **Alex**: Can you deploy payments-api to Development? ✅ Can you deploy to Production? ❌ Can you run a runbook? ✅ Can you see the Merchants space? (Depends on your config)
- As **Sarah**: Can you do everything across all spaces? ✅

### Install K8s Agents

Each space needs its own K8s Agent per cluster. Find where deployment targets are managed and add a Kubernetes Agent for each combination below:

| Space | Cluster Context | Agent Name | Target Tag | Environments |
|-------|----------------|------------|------------|-------------|
| Platform | `kind-finpay-dev` | `platform-dev` | `platform-k8s` | Development, Staging |
| Platform | `kind-finpay-prod` | `platform-prod` | `platform-k8s` | Production |
| Payments | `kind-finpay-dev` | `payments-dev` | `payments-k8s` | Development, Staging |
| Payments | `kind-finpay-prod` | `payments-prod` | `payments-k8s` | Production |
| Merchants | `kind-finpay-dev` | `merchants-dev` | `merchants-k8s` | Development, Staging |
| Merchants | `kind-finpay-prod` | `merchants-prod` | `merchants-k8s` | Production |

After all 6 agents are installed, check the footprint:

```bash
echo "Dev cluster pods:       $(kubectl --context kind-finpay-dev get pods -A | grep -c octopus)"
echo "Dev cluster namespaces: $(kubectl --context kind-finpay-dev get ns | grep -c octopus)"
echo "Prod cluster pods:      $(kubectl --context kind-finpay-prod get pods -A | grep -c octopus)"
echo "Prod cluster namespaces:$(kubectl --context kind-finpay-prod get ns | grep -c octopus)"
```

---

*The stage is set. Now the real work begins.*

---

## Chapter 1: "We Need payments-api in Staging by Lunch"

**Monday morning.** Marcus's team merged a new refund flow into `payments-api`. QA wants it in staging by lunch, and the risk team needs to sign off before it goes to production on Wednesday.

### What You'll Do

Switch to the **Payments** space.

**Goal:** Create a `payments-api` project that deploys the Helm chart from your finpay-deploy repo, with environment-specific values files, and a production approval gate.

**Key details:**
- The chart is at `charts/payments-api/` in your repo
- Each environment has its own values file (`values-development.yaml`, `values-staging.yaml`, `values-production.yaml`) — figure out how to dynamically pick the right one based on the environment
- The namespace is different per environment — use a scoped project variable:

| Variable | Value | Scope |
|----------|-------|-------|
| `Namespace` | `payments-dev` | Development |
| `Namespace` | `payments-staging` | Staging |
| `Namespace` | `payments-prod` | Production |

- Production deploys require an approval gate so the risk team can sign off — how would you enforce this in the deployment process?
- Use the "Standard" lifecycle
- Target tag: `payments-k8s`

**Also:** Include the "Common Config" library variable set in this project.

Once the project is configured:

1. Create Release v1.0.0
2. Let it auto-deploy to Development
3. Promote to Staging
4. Promote to Production and go through the approval flow

```bash
# Verify
kubectl --context kind-finpay-dev get pods -n payments-dev
kubectl --context kind-finpay-dev get pods -n payments-staging
kubectl --context kind-finpay-prod get pods -n payments-prod
```

### 📡 Live Object Status

Before moving on — switch the project dashboard from deployment status to **live status** (there's a toggle). This uses a monitor component running inside the K8s Agent that continuously watches the objects Octopus deployed.

Drill into the payments-api deployment in Development:
- What objects does Octopus know about? (Deployment, Service, Pods, HPA, PDB...)
- Check the **events** for a pod — compare with `kubectl describe pod`
- Check the **logs** — compare with `kubectl logs`
- Look at the **live manifest** — Octopus shows what's actually running in the cluster, not just what it applied

Now simulate someone making a manual change:

```bash
kubectl --context kind-finpay-dev set image deployment/payments-api \
  payments-api=nginx:1.24-alpine -n payments-dev
```

Check the live status again. What changed? Does Octopus detect the drift?

Revert it:

```bash
kubectl --context kind-finpay-dev set image deployment/payments-api \
  payments-api=nginx:1.25-alpine -n payments-dev
```

### 📝 What to Notice

- How many clicks from "code merged" to "running in staging"?
- Can you see in the deployment log what Helm commands Octopus actually ran?
- The release `v1.0.0` is an immutable snapshot that flows through environments — how does that compare to GitOps where each environment independently tracks Git?
- Live Object Status is powered by the Kubernetes Monitor — a separate component that ships alongside the K8s Agent. The Monitor uses the Argo GitOps Engine library under the hood (the same engine ArgoCD uses for reconciliation)

---

## Chapter 2: "Fraud-detector Is Acting Up in Staging"

**Tuesday afternoon.** A Slack message from the fraud team: *"fraud-detector is returning false positives on every transaction in staging. Can someone restart it?"*

Junior developer Alex doesn't have `kubectl` access to the staging cluster — FinPay restricts direct cluster access to the Platform team. But Alex should be able to restart a service in staging through Octopus.

### What You'll Do

**Part A:** Create and deploy the `fraud-detector` project in the Payments space. Same pattern as payments-api — Helm chart from your repo (`charts/fraud-detector/`), environment-scoped namespaces:

| Variable | Value | Scope |
|----------|-------|-------|
| `Namespace` | `fraud-dev` | Development |
| `Namespace` | `fraud-staging` | Staging |
| `Namespace` | `fraud-prod` | Production |

Deploy through to Staging.

**Part B:** Create a "Restart Service" runbook on the fraud-detector project. The runbook should:
- Run a `kubectl rollout restart` against the deployment in the appropriate namespace
- Use `#{Namespace}` variable so it works for any environment
- Be runnable by Alex (junior dev) — verify the RBAC you set up allows this

**Part C:** Switch to the Platform space. Create a `cluster-ops` project with a "Health Check" runbook. This should report node status, unhealthy pods, and recent warning events. It runs against the Platform agent.

Run the restart runbook as "Alex" against Staging. Run the health check from the Platform space.

### 📝 What to Notice

- The restart runbook asks you to pick an environment. Does this make sense for all runbook types?
- Alex never got `kubectl` access, but could restart the service. Where is this recorded?
- The Platform space health check runs on a different agent that happens to be on the same cluster as the Payments agent. They're completely independent in Octopus.

---

## Chapter 3: "KYC-Service Needs to Launch — and It Handles PII"

**Wednesday.** Priya's Merchant team built `kyc-service` — it processes identity documents for merchant onboarding. It handles PII (personally identifiable information), so compliance requires explicit approval before ANY staging or production deployment.

KYC-service doesn't use Helm — the team prefers raw Kubernetes YAML with Octopus variable substitution.

### What You'll Do

Switch to the **Merchants** space.

**Goal:** Create a `kyc-service` project that deploys raw YAML from `manifests/kyc-service/deployment.yaml` in your repo.

**Key details:**
- The YAML uses `#{VariableName}` syntax (Octopus variable substitution) — look at the file to see which variables you need to define
- Environment-scoped variables:

| Variable | Dev | Staging | Prod |
|----------|-----|---------|------|
| `Namespace` | `kyc-dev` | `kyc-staging` | `kyc-prod` |
| `Replicas` | `1` | `2` | `2` |
| `LogLevel` | `debug` | `info` | `warn` |
| `DocumentStorageUrl` | `s3://finpay-dev-kyc-docs` | `s3://finpay-staging-kyc-docs` | `s3://finpay-prod-kyc-docs` |
- Compliance requires a Manual Intervention step for Staging AND Production (not just prod)
- Use the "Standard" lifecycle
- Target tag: `merchants-k8s`

Deploy through to Staging (approve the compliance sign-off).

### 📝 What to Notice

- Compare what's in Git (`replicas: #{Replicas}`) with what's running in the cluster (`kubectl get deployment kyc-service -n kyc-staging -o yaml | grep replicas`). What's in Git is NOT what's running.
- Raw YAML required more Octopus variables than Helm did (Helm puts these in values files). What are the tradeoffs?
- Check the **live status** for kyc-service — does it track raw YAML deployments the same way it tracked Helm deployments in Chapter 1?

---

## Chapter 4: "EuroShop Just Signed — Spin Up Their Environment"

**Thursday.** Sales closes EuroShop, a European merchant. FinPay hosts the merchant portal per-customer, each with isolated config (branding, data region, API keys). The existing customer is Acme Corp.

Priya's team needs to deploy `merchant-portal` for both merchants from the same codebase but with different configuration.

This is what Octopus **Tenants** are for.

### What You'll Do

Still in the **Merchants** space.

**Goal:** Create a tenanted `merchant-portal` project using the Helm chart at `charts/merchant-portal/`.

**Key details:**
- Create two tenants: `Acme Corp` and `EuroShop`
- Connect both tenants to the project and appropriate environments
- Define tenant-scoped variables: each tenant gets its own namespace, brand color, and data region per environment:

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
- The Helm release name must include the tenant name to avoid collisions
- The tenant-specific values (brand color, data region) need to reach the running application — figure out how to pass Octopus variables as Helm value overrides
- Deploy both tenants to Development

After deploying, check the project overview page for the tenant dashboard — the matrix of tenants × environments.

**This project should be stored in Git (Config-as-Code).** Octopus supports storing project configuration in a Git repo instead of the platform database. Figure out how to enable this when creating the project, point it to your finpay-deploy fork, and explore what ends up in the repo. Try making a change on a branch — what's the workflow for testing deployment process changes?

### 📝 What to Notice

- The variable matrix: 2 tenants × 3 environments × N variables. How would this scale to 50 merchants?
- The tenant dashboard — is it useful? What questions does it answer easily?
- Config-as-Code: look at the `.ocl` files in your repo. How readable are they? Could you review a PR that changes the deployment process?

---

## Chapter 5: "New Image Pushed — Auto-Deploy to Dev"

**Friday morning.** FinPay's CI pipeline (GitHub Actions) builds and pushes a new `payments-api` container image after every merge to main. Marcus doesn't want to manually create a release every time — it should happen automatically.

You'll simulate the CI pipeline by tagging and pushing images to your own Docker Hub repo, then configure Octopus to detect new tags and auto-deploy.

### Simulate the CI Pipeline

FinPay's CI pushes images to `<org>/finpay-payments-api` on Docker Hub. You'll simulate this with your own Docker Hub account.

Log in and push an initial "build" — just nginx re-tagged under your namespace:

```bash
docker login

export DOCKERHUB_USER=<your-dockerhub-username>
docker tag nginx:1.25-alpine $DOCKERHUB_USER/finpay-payments-api:1.0.0
docker push $DOCKERHUB_USER/finpay-payments-api:1.0.0
```

This represents the current production image that CI has already built and pushed.

### What You'll Do

Switch to the **Payments** space.

**Goal:** Configure the `payments-api` project so Octopus detects new image tags pushed to your Docker Hub repo, auto-creates a release, and deploys to Development.

**You need to solve four things:**
1. The Docker Hub feed you set up earlier can only see public images — update it so Octopus can query *your* repository
2. The payments-api deployment process needs to know which container image to track. Find how to add a package reference to the Helm step
3. The Helm chart supports `image.repository` and `image.tag` — override these with the values from Octopus's package tracking
4. Set up an automatic trigger so new image versions create releases without manual intervention

### Test It

Push a new "build":

```bash
docker tag nginx:1.25-alpine $DOCKERHUB_USER/finpay-payments-api:1.1.0
docker push $DOCKERHUB_USER/finpay-payments-api:1.1.0
```

Octopus polls the feed periodically (usually every 1-2 minutes). Watch the project dashboard — a new release should appear and auto-deploy to Development.

```bash
# Verify the new image version is running
kubectl --context kind-finpay-dev get pods -n payments-dev \
  -o jsonpath='{.items[*].spec.containers[*].image}' && echo
```

**Also:** Explore **Channels**. Octopus can route releases through different lifecycles based on rules. Create a way for hotfix releases to skip dev and go straight to staging → prod. How would you automatically route image tags like `*-hotfix` to a different path?

#### One More Thing

Look at the release that was just auto-created. What's its version number? Now look at the Docker image tag you pushed. Do they match?

Explore **Settings → Release Versioning** for the payments-api project. There are three approaches — what happens to your auto-created releases if you switch from the default to tying the release version to the package you just referenced?

Push another image tag and watch what changes.

### 📝 What to Notice

- Release `0.0.3` was triggered by image `1.1.0` — would a new team member understand that? What release versioning strategy makes auto-deploy releases self-documenting?
- How long between `docker push` and Octopus creating the release? The feed is polled, not pushed — what are the implications at scale?
- When the trigger fires, what gets captured in the release? Is it clear which image version the release contains?
- The channel + lifecycle combination — is it intuitive? How would you explain it to a new team member?
- Compare this to a webhook-driven approach (CI calls the Octopus API directly after a push). What are the tradeoffs of polling vs. pushing?

---

## Chapter 6: "The Compliance Audit"

**The following Monday.** FinPay's SOC 2 auditor shows up (remotely). They want answers to specific questions about production deployment controls. Every fintech deals with this.

### The Auditor's Questions

Work through these using the Octopus UI. Document your findings in your Google Doc.

1. **"Show me every production deployment in the last 30 days."** — Can you get a cross-space view? Does the live status dashboard help here?
2. **"Who approved each production deployment?"** — Where do you find the approver? Is it in the summary or buried in step details?
3. **"What changed between the staging deploy and the production deploy of payments-api?"** — Can you diff what was applied?
4. **"Show me that kyc-service cannot be deployed to production without approval."** — Is this enforced or just configured? Could an admin bypass it?
5. **"Can a developer in the Payments team deploy to the Merchant team's production?"** — Test the RBAC you configured. Does space isolation prevent this? What about at the K8s level?
   - **Bonus:** Think about whether per-person deploy restrictions (like Alex's) actually make sense when multiple developers' code ships in the same release. Is Octopus RBAC the right layer for this, or should it be handled by PR approvals and pipeline gates?
6. **"Show me all operational actions (restarts, health checks) run against production."** — Is there a unified view across projects and spaces?

---

## Chapter 7: "Sarah Wants to Try the ArgoCD Path"

**Two weeks later.** Sarah (Platform lead) has been watching the Payments team's experience. The native Helm deployment works, but she wants to evaluate Octopus's ArgoCD integration as an alternative. The idea: Octopus handles promotion and approval, ArgoCD handles the actual cluster sync.

### What You'll Do

#### Step 1: Install ArgoCD on the Dev Cluster

```bash
kubectl config use-context kind-finpay-dev

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server \
  -n argocd --timeout=180s

# Get the admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Port-forward for UI access
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
```

#### Step 2: Generate an ArgoCD API Token

The `admin` account can't generate tokens by default — you need to enable the `apiKey` capability first:

```bash
# Patch the argocd-cm ConfigMap to allow token generation for admin
kubectl -n argocd patch configmap argocd-cm --type merge -p '{"data":{"accounts.admin":"apiKey, login"}}'

# Restart the server to pick up the change
kubectl -n argocd rollout restart deployment argocd-server
kubectl -n argocd wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server --timeout=120s
```

Now generate the token:

```bash
brew install argocd  # if you don't have it already

argocd login localhost:8080 --insecure --username admin --password <password-from-above>
argocd account generate-token --account admin
```

#### Step 3: Install the Octopus ArgoCD Gateway

In the **Payments** space, find where ArgoCD instances are managed and add one. The wizard walks you through naming, environment selection, ArgoCD connection details, and generates a Helm command.

After installing, compare the footprint:

```bash
echo "ArgoCD Gateway pods:"
kubectl --context kind-finpay-dev get pods -A | grep argo-gateway
echo ""
echo "K8s Agent pods (for comparison):"
kubectl --context kind-finpay-dev get pods -A | grep octopus-agent
```

#### Step 4: Create ArgoCD Applications

The finpay-deploy repo includes `argocd-manifests/applications.yaml`. Update the repo URL to point to your fork, commit, and apply:

```bash
cd ~/finpay-deploy
# Edit argocd-manifests/applications.yaml — replace <YOUR_USERNAME> with your GitHub username
git add -A && git commit -m "Set repo URL in ArgoCD applications" && git push
kubectl --context kind-finpay-dev apply -f argocd-manifests/applications.yaml
```

Look at the `argo.octopus.com/*` annotations in that file — they're the glue between ArgoCD and Octopus.

> The applications file only covers Development and Staging — both namespaces live on the dev cluster where ArgoCD is installed. In production, you'd run a separate ArgoCD instance on the prod cluster.

#### Step 5: Create an Octopus Project (ArgoCD Mode)

Create a `payments-api-argo` project in the Payments space. Instead of deploying Helm directly, this project should update the ArgoCD application's image tags — find the right step type for this. Deploy and observe the flow: how does the change get from Octopus to the cluster?

#### Step 6: Test Drift Detection

> ⚠️ **Namespace overlap:** If your Chapter 1 native Helm deployment of `payments-api` is still running in `payments-dev`, the ArgoCD Application now targets the same deployment. Either delete the Ch1 release first (`helm uninstall payments-api -n payments-dev --kube-context kind-finpay-dev`) or be aware that both Octopus and ArgoCD are now managing the same object — which is itself an interesting learning moment about ownership conflicts.

Simulate someone making a manual change:

```bash
kubectl --context kind-finpay-dev set image deployment/payments-api \
  payments-api=nginx:1.24-alpine -n payments-dev
```

Watch ArgoCD revert it. This is the reconciliation loop that native Octopus doesn't have.

### 📝 What to Notice

- Gateway (1 pod) vs Agent (3+ pods) — what's the tradeoff?
- The model shift: Octopus commits to Git instead of running Helm directly. What does Octopus add on top of ArgoCD alone?
- Could the Merchants space use the same Gateway and ArgoCD instance, or would they need their own?
- Check the **live status** for `payments-api-argo` — does Octopus track the objects that ArgoCD deployed, or only objects it deployed directly? Compare this with the live status you saw for the native Helm `payments-api` project in Chapter 1. What are the visibility tradeoffs between the two paths?

---

## Chapter 8: "The Database Migration"

**Wednesday of week 3.** The Payments team needs to run a database migration before deploying payments-api v1.2.0. Migrations are risky — they need to be trackable, approvable, and never accidentally run twice.

### What You'll Do

Switch to the **Payments** space.

**Goal:** Create a "Run DB Migration" runbook on the payments-api project.

**Key details:**
- The runbook should accept a **prompted variable** — the migration name (e.g., `V1.2__add_refund_column`), entered at runtime
- A Manual Intervention step should gate staging and production runs
- The script step should log the migration name, environment, who ran it, and timestamp
- A verification step should confirm the service is healthy after the migration
- Run it against Development, then Staging (observe the approval flow)

Check the runbook run history when done — each run captures the migration name, operator, environment, and timestamp.

---

## Chapter 9: "Git-Driven Variables"

**Thursday of week 3.** Marcus's team is tired of managing `fraud-detector` variables through the Octopus UI. They want to manage them in Git alongside the code, reviewable in PRs.

### What You'll Do

**Goal:** Reconfigure the `fraud-detector` project to use Git-sourced variables instead of platform-managed variables.

Explore how to:
- Move existing project variables into the Git repository (`.ocl` files)
- Change variable values through a Git commit + PR instead of the Octopus UI
- Understand which variable types CAN live in Git and which must stay in the platform (hint: what about sensitive variables like secrets?)

Make a change to a variable via a Git commit and verify it takes effect on the next deployment.

### 📝 What to Notice

- Which variables were easy to move to Git? Which couldn't be moved?
- How does the PR review experience work for variable changes?
- If a variable is wrong, how do you roll back — Git revert, or Octopus UI, or both?

---

## The Retrospective

You've lived through 3 weeks at FinPay. Step back and capture your experience.

### Footprint Check

```bash
echo "=== TOTAL OCTOPUS FOOTPRINT ==="
echo "Dev cluster pods:       $(kubectl --context kind-finpay-dev get pods -A | grep -c octopus)"
echo "Dev cluster namespaces: $(kubectl --context kind-finpay-dev get ns | grep -c octopus)"
echo "Dev cluster PVCs:       $(kubectl --context kind-finpay-dev get pvc -A | grep -c octopus)"
echo "Prod cluster pods:      $(kubectl --context kind-finpay-prod get pods -A | grep -c octopus)"
echo "Prod cluster namespaces:$(kubectl --context kind-finpay-prod get ns | grep -c octopus)"
echo "Prod cluster PVCs:      $(kubectl --context kind-finpay-prod get pvc -A | grep -c octopus)"
```

### Tally

| What | Count |
|------|-------|
| Times you created "Development/Staging/Production" environments | |
| Times you created the "Standard" lifecycle | |
| Times you stored the same Git credential | |
| Times you created the "Common Config" variable set | |
| Total K8s Agent installations | |
| Places you'd need to update if the Kafka broker address changes | |

### Open Questions

Reflect on these — there are no right answers:

- What would change if FinPay had 10 spaces instead of 3? 50?
- Which Octopus concepts mapped cleanly to things you already know? Which felt foreign?
- Where did you feel like the platform was helping you vs. getting in the way?
- How would you onboard a new engineer to this setup?
- If you were starting from scratch, would you choose the native Helm path or the ArgoCD path? For which use cases?
- What would you want to build or change about this setup?
- You set up 3 spaces manually — creating the same environments, lifecycles, credentials, and variable sets each time. How could you automate space bootstrapping? (Octopus has a [REST API](https://octopus.com/docs/octopus-rest-api) and a community [Terraform provider](https://registry.terraform.io/providers/OctopusDeployLabs/octopusdeploy/latest) — at what scale does automation become worth the investment?)

### Your Notes

Transfer the key insights from your Google Doc into your summary. Focus on what surprised you — things that were better than expected and things that were worse.

---

## Cleanup

```bash
kind delete cluster --name finpay-dev
kind delete cluster --name finpay-prod
rm -rf ~/finpay-deploy
```

---

