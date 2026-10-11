---
title: Adding a cloud adapter
---

# Adding a Cloud Adapter

This walks through adding support for a new cloud, following the pattern used by the 4 container adapters already in `packages/adapters/src` (one per cloud: AWS ECS Fargate, Azure Container Apps, GCP Cloud Run, on-prem Docker via SSH). AbstractAI supports a single deployment strategy — an image registry plus each cloud's native container-hosting service — so a new adapter targets that same shape rather than introducing a new strategy.

## 1. Create the adapter class

Extend `BaseAdapter` (`packages/adapters/src/base-adapter.ts`), which gives you:

- `this.stackName(agentName, region)` — the standard Pulumi stack-naming convention (`agent-platform-{agent}-{cloud}-{strategy}-{region}`)
- `this.createStack(config, program)` / `this.getStack(stackName)` / `this.destroyStack(stackName)` — Pulumi Automation API stack lifecycle
- Default `getSchema()` / `validate()` via the core `SchemaEngine` (cloud/strategy mismatch + `cloudOptions` schema validation)

```ts
export class MyCloudContainersAdapter extends BaseAdapter {
  constructor(vault: VaultClient) {
    super(vault, CloudProvider.MYCLOUD, DeploymentStrategy.CONTAINERS)
  }
  // implement plan/deploy/status/destroy/logs/scale — see below
}
```

> `cloud`/`strategy` are passed to `super()`, not declared as class fields — a base-class constructor can't read a subclass's field initializer (JS runs the base constructor to completion first).

## 2. Implement all 8 `CloudAdapter` methods

`getSchema` and `validate` come from `BaseAdapter` by default (override only if you need cloud-specific validation beyond the JSON schema). You must implement:

- **`plan(config)`** — returns an `ExecutionPlan` (steps + planned resources). Pure and side-effect-free; don't call Pulumi here.
- **`deploy(config, plan)`** — builds a Pulumi inline program (or, for non-Pulumi targets like the on-prem SSH-based adapters, does the deployment directly) and returns a `DeploymentResult`. The `deploymentId` you return here becomes the string every other method receives — see the note on `deploymentId` below.
- **`status(deploymentId)`**, **`destroy(deploymentId)`**, **`logs(deploymentId, options?)`**, **`scale(deploymentId, replicas)`** — reconnect using only that one string.

Dynamically import cloud SDKs (`@aws-sdk/*`, `@google-cloud/*`, `@azure/*`, `node-ssh`) rather than adding them as hard dependencies — use `importOptionalSdk()` from `base-adapter.ts`, which throws a clear "install X" error if the package isn't present. `@pulumi/*` packages, by contrast, **are** regular dependencies (every adapter needs at least one of them).

### The `deploymentId` convention

`status`/`destroy`/`logs`/`scale` receive only a single string — no config, no region — so it has to be self-describing. The convention here is that it's the Pulumi stack name (parseable via `parseStackName()`), or, for adapters that don't use Pulumi at all (see the on-prem containers adapter), a base64url-encoded token carrying whatever connection info is needed (SSH host/user/project, etc.). The orchestrator (`packages/core/src/orchestrator/deployment-orchestrator.ts`) is what translates between this and the database's own deployment id — it stores whatever `deploy()` returns as `deploymentId` into the `Deployment.pulumiStackName` column, and always hands callers the database id instead.

## 3. Create a JSON Schema for `cloudOptions`

Add `packages/core/src/schema/schemas/{cloud}/containers.json` — draft-07 JSON Schema with `$id`, `type: "object"`, `properties` (each with a `description`), and `required`. This is what `BaseAdapter.validate()` checks automatically. Look at an existing schema (e.g. `aws/containers.json`) for the shape.

## 4. Register the adapter

Add it to `packages/adapters/src/register.ts`'s `registerAllAdapters()`, and to `packages/adapters/src/index.ts`'s exports.

## 5. Add IAM mappings if needed

If your cloud needs a new entry in the abstract-permission table, add it to `PERMISSION_TABLE` in `packages/core/src/security/iam-mapper.ts`, and extend `generatePolicy()` with a case for your cloud's native policy document format.

## 6. Write tests

At minimum, mirror the pattern in `packages/adapters/src/__tests__/gcp-containers.test.ts`:

- `validate()` accepts a valid config and rejects a mismatched cloud or missing required `cloudOptions`.
- `plan()` returns the expected steps/resources — this needs no live cloud credentials since `plan()` is pure.
- `stackName()` follows the naming convention.

Don't test `deploy()`/`status()`/`logs()`/`scale()` against real infrastructure in the unit suite — those need a mocked adapter registered on a `PluginRegistry` at the orchestrator/router level instead (see `packages/core/src/__tests__/orchestrator.test.ts`).

## 7. Update documentation

Add your cloud/strategy combination to the tables in `README.md` (features list, config reference) if it changes what's publicly supported.
