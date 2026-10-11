---
title: Deploying an agent
---

# Deploying an agent with AbstractAI

This guide walks through deploying your own AI agent to a cloud of your choice using the
AbstractAI CLI. It covers what's common to every cloud, then a cloud-specific section for each
one that has been deployed and verified end-to-end.

## Architecture in one paragraph

AbstractAI has two separate things: the **control plane** (the AbstractAI API + web app itself —
one shared instance, run by whoever operates AbstractAI) and your **agent** (the thing you deploy
*through* the control plane, onto whichever cloud you target). When you run `agent-platform
deploy`, two steps happen in two different places: the CLI builds and pushes your container image
**from your own machine**, using your own cloud credentials; then it calls the control plane's
API, which runs Pulumi **on the control plane** to actually create the cloud resources (using the
control plane's own stored credentials for that cloud). You only ever touch step one's
credentials — the control plane's credentials are configured once by whoever runs AbstractAI.

## Prerequisites (every cloud)

1. **Node.js 20+** installed locally (the CLI runs as a Node package via `npx`).
2. **The AbstractAI CLI reaches the control plane.** There's no CLI prompt for this — set it
   either as an env var for the current shell:
   ```bash
   # PowerShell
   $env:AGENT_PLATFORM_API_URL = "http://<control-plane-address>:4000/trpc"
   ```
   or persist it by editing `~/.agent-platform/config.json` directly (create the file/folder if
   it doesn't exist yet):
   ```json
   { "apiUrl": "http://<control-plane-address>:4000/trpc" }
   ```
   Then log in, which stores an auth token in the same file:
   ```bash
   agent-platform login
   ```

   > **Known limitation:** this testing deployment has no load balancer or DNS name yet — the
   > control plane's address is a raw IP that changes on every control-plane redeploy, so you'll
   > need to repeat this step (and re-run `login`) each time the operator redeploys the control
   > plane. Whoever operates AbstractAI needs to tell you the current address until an ALB + DNS
   > name is in place. Once that's done, this becomes a true one-time step, and ideally the CLI
   > should grow a real `agent-platform configure` command instead of requiring a hand-edited
   > JSON file.
3. **Create an account**, if you haven't:
   ```bash
   agent-platform signup
   ```
4. **Add a cloud account** (bookkeeping only — see note below):
   ```bash
   agent-platform accounts add
   ```
   This asks which cloud, a name/region, and — depending on cloud — some credentials. For AWS/GCP
   it prompts for real credentials; for Azure it also asks for credentials, but **note**: as of
   this writing, the orchestrator does not actually use these stored credentials to deploy — every
   deploy uses the control plane's own cloud identity, regardless of which account you select here.
   The account record is still required (every deploy needs an `--account` or default account ID
   for bookkeeping/audit logging), but don't expect per-user cloud isolation yet.
5. **Write an `agent.yaml`** (or run `agent-platform init` to generate one interactively).

With that done, the cloud-specific sections below cover what else is needed.

---

## Deploying to AWS

### One-time setup (done by whoever operates the control plane)
Nothing extra — AWS deploys use the control plane's own ECS task IAM role (`ApiTaskRole`) to
create resources directly. No per-user AWS credentials are required on the control-plane side.

### Your `agent.yaml`
```yaml
name: my-agent
version: 1.0.0
runtime: node20
cloud: aws
region: us-east-1
strategy: containers
source:
  type: local
  path: .
resources:
  memory: 512Mi
  cpu: 0.25
networking:
  public: true
cloudOptions:
  clusterName: my-agent-cluster
env:
  GEMINI_API_KEY: "..."
```

### Deploy
```bash
agent-platform deploy
```
The CLI builds your image via **AWS CodeBuild** (no local Docker needed — this still runs from
your machine using your own local AWS credentials, same as every cloud's build step), pushes it
to ECR, then the control plane creates an ECS Fargate service for it.

### After deploy
There's no load balancer — each task gets its own public IP that changes if the task restarts.
Find it with:
```bash
aws ecs list-tasks --cluster my-agent-cluster --service-name my-agent --region us-east-1
# then describe-tasks + describe-network-interfaces to get the public IP (see README for the full chain)
```

---

## Deploying to Azure

Azure needs more one-time setup than AWS, because the control plane has no native trust
relationship with Azure the way it does with its own AWS IAM role — it needs an actual Azure
identity to act on your behalf.

### One-time setup (done once per Azure subscription, by whoever operates the control plane)

1. **Register required resource providers** (fresh subscriptions don't have these enabled):
   ```bash
   az provider register --namespace Microsoft.App
   az provider register --namespace Microsoft.ContainerRegistry
   az provider register --namespace Microsoft.OperationalInsights
   az provider register --namespace Microsoft.ManagedIdentity
   az provider register --namespace Microsoft.Authorization
   ```
2. **Create a resource group:**
   ```bash
   az group create --name my-azure-rg --location eastus
   ```
3. **Create a Container Apps environment** (the adapter only *references* one by name — it never
   creates it, so this must exist before any agent can deploy into it):
   ```bash
   az containerapp env create \
     --name my-agent-env \
     --resource-group my-azure-rg \
     --location eastus \
     --enable-workload-profiles false
   ```
   (`--enable-workload-profiles false` matters: it keeps the environment compatible with the
   Pulumi `azure-native` provider version this codebase currently pins.)
4. **Create a service principal** for the control plane to use, scoped to just that resource
   group:
   ```bash
   az ad sp create-for-rbac --name abstractai-azure-deployer --role Contributor \
     --scopes /subscriptions/<sub-id>/resourceGroups/my-azure-rg
   ```
   Capture `appId`, `password`, `tenant` — don't paste them anywhere they'd get logged.
5. **Grant that service principal `User Access Administrator`** on the same resource group. This
   is a separate step because `Contributor` explicitly excludes
   `Microsoft.Authorization/roleAssignments/write`, and the control plane needs to grant each
   agent's identity pull access to its own container registry:
   ```bash
   az role assignment create --assignee <appId> --role "User Access Administrator" \
     --scope /subscriptions/<sub-id>/resourceGroups/my-azure-rg
   ```
6. **Give the control plane those credentials** as `ARM_CLIENT_ID` / `ARM_CLIENT_SECRET` /
   `ARM_TENANT_ID` / `ARM_SUBSCRIPTION_ID` env vars (plus the `AZURE_*`-named equivalents, which
   the build pipeline reads directly). In this repo that's wired through CloudFormation
   parameters → GitHub secrets → the control plane's ECS task definition — see
   `infra/platform/template.yaml` and `.github/workflows/deploy-dev.yml`.

### Your own local prerequisites (every time you deploy, not one-time)
Azure's image build step runs on **your** machine, so you need:
```bash
az login
```
and (in the same shell you'll run `agent-platform deploy` from):
```bash
# PowerShell
$env:AZURE_SUBSCRIPTION_ID = "<sub-id>"
```

### Your `agent.yaml`
```yaml
name: my-agent
version: 1.0.0
runtime: node20
cloud: azure
region: eastus
strategy: containers
source:
  type: local
  path: .
resources:
  memory: 512Mi    # must convert to one of Azure's fixed cpu/memory pairs — see note below
  cpu: 0.25
networking:
  public: true
cloudOptions:
  resourceGroup: my-azure-rg
  environmentName: my-agent-env
env:
  GEMINI_API_KEY: "..."
```

> **Resource sizing note:** Azure Container Apps' Consumption plan only accepts a fixed list of
> cpu/memory pairs (0.25 cpu ↔ 0.5Gi, 0.5 ↔ 1.0Gi, 0.75 ↔ 1.5Gi, ... up to 4.0 ↔ 8Gi). Pick a
> `memory`/`cpu` combo in `agent.yaml` that converts to one of these pairs (512Mi/0.25 → 0.5Gi,
> which is valid).

### Deploy
```bash
agent-platform deploy
```
This builds your image via **Azure Container Registry Quick Tasks** (from your machine, using
your `az login` session), then the control plane:
1. Creates a dedicated **user-assigned managed identity** for your agent
2. Grants it **AcrPull** on your ACR registry (the registry has no admin user — identity-based
   pulls only)
3. Creates the **Container App**, configured to pull using that identity

### After deploy
Unlike AWS, Azure gives you a stable HTTPS hostname immediately:
```bash
az containerapp show --name my-agent --resource-group my-azure-rg \
  --query "properties.configuration.ingress.fqdn" --output tsv
```
or just read it from the CLI's own deploy output (`endpoint` field).

---

## GCP and on-prem

Adapters exist for both (`gcp-containers.adapter.ts`, `onprem-containers.adapter.ts`) and
`agent-platform accounts add` supports both, but neither has been exercised through a live,
end-to-end deploy the way AWS and Azure have in this guide. Expect an analogous pattern — GCP via
a service-account JSON key and Cloud Build, on-prem via SSH/kubeconfig — but budget time for
discovering real bugs the first time through, the same way AWS and Azure needed several rounds of
live fixes before they worked cleanly.

---

## Known platform limitations worth knowing about

- **No stable control-plane address.** Every control-plane redeploy changes its IP; there's no
  ALB/DNS in front of it yet.
- **Cloud accounts aren't actually used for credentials.** The orchestrator always deploys using
  the control plane's own ambient cloud identity, never the credentials you enter in
  `accounts add`. There's no real per-user cloud isolation yet.
- **Pulumi's state is ephemeral.** It lives in the control plane container's own filesystem, not
  a persistent backend like S3 — a control-plane restart can desync Pulumi's view of what already
  exists in your cloud account, surfacing as "resource already exists" errors that need manual
  cleanup.
