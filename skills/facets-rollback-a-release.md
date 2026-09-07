---
name: Roll a Facets release back to a known-good state
description: Reverse a bad release using the two-phase Facets rollback - create a ROLLBACK_PLAN that changes nothing, review it, then apply it - including how to check a release is even a valid rollback target.
api: openapi/facets-control-plane-openapi.yml
operations:
  - getDeployments
  - getDeployment
  - getLatestRelease
  - triggerRollbackPlanRelease
  - getReleaseChanges
  - getDeploymentLogs
  - abortRelease
generated: '2026-09-07'
method: generated
source: Grounded in operationIds verified verbatim in openapi/_original/facets-control-plane-openapi.json, plus https://www.facets.cloud/docs/features-and-guides/releases-concept/rolling-back-a-release.
---

# Roll a Facets release back to a known-good state

Rollback in Facets is **two phases**: creating the plan changes nothing on the environment;
a separate apply executes it. Both are recorded as separate releases in the history, so the
plan you reviewed and the apply that acted on it stay auditable.

## First, decide whether rollback is even the right move

- If the bad release is **still running**, do not roll back — abort it. `abortRelease` —
  `POST /cc-ui/v1/clusters/{clusterId}/deployments/{deploymentId}/abort`.
- If it has **finished**, continue here.

## Not every release can be rolled back to

A release is only a valid rollback target if it has a `deploymentContextFilePath` stored in
S3. Facets states that releases created before the deployment-context storage feature shipped
cannot be rollback targets, and the UI will prompt for a more recent one.

**Facets does not publish how long that context is retained.** Do not tell a human "you can
roll back for N days" — no such window is documented. Tell them what you can actually verify:
which specific releases in the history carry a `deploymentContextFilePath`.

## Steps

1. **Find the candidate.** `getDeployments` —
   `GET /cc-ui/v1/clusters/{clusterId}/deployments` for the environment's release history, or
   `getLatestRelease` — `.../deployments/latest-successful-release` for the most recent
   success. Read each candidate with `getDeployment` —
   `GET /cc-ui/v1/clusters/{clusterId}/deployments/{deploymentId}`.

2. **Verify it is a valid target.** On the candidate `DeploymentLog`, confirm
   `deploymentContextFilePath` is present and `status` is a success. That file holds the full
   state — `stackSourceVersion`, overrides, `resourceMetadata`, modules and resources — that
   the rollback restores.

3. **Create the plan.** `triggerRollbackPlanRelease` —
   `POST /cc-ui/v1/clusters/{clusterId}/deployments/{deploymentId}/{resourceType}/{resourceName}/rollback-plan`.
   Note the scope: a rollback plan is **per resource**, not per environment. `deploymentId`
   is the release you are rolling back *to*.

   This creates a release of type `ROLLBACK_PLAN`. **It changes nothing.**

4. **Review the plan and show it to the human.** Read it with `getReleaseChanges` —
   `.../release-changes` and `getDeploymentLogs` — `.../logs`. Surface, explicitly:
   - resources that will be restored,
   - stack versions that will be applied,
   - configuration differences between current and previous state,
   - **any resource the plan will destroy or replace** — say this first, in plain words.

5. **Apply only after explicit confirmation.** The apply is performed from the
   `ROLLBACK_PLAN` release in the environment's release history. Facets records the result as
   a separate `APPLY ROLLBACK PLAN` release. If the environment has a release approval gate
   configured, the apply passes through it like any other release.

6. **Verify.** Confirm the `APPLY ROLLBACK PLAN` release finished `SUCCEEDED`, read its
   Terraform logs to confirm the changes the plan listed are the changes that ran, and
   re-read the environment's resources with `getResourceByNameV2`.

## Related reversals

- `restore` — `POST /cc-ui/v1/versions/{versionId}/restore` restores a previous blueprint
  version. Different thing: this reverses a *configuration* change, not a deployment.
- `restoreSoftDelete` — `POST /cc-ui/v1/versions/softDeletedEntities/{entityId}` restores a
  soft-deleted entity. No retention period is published; check whether the entity is still
  listed before promising anything.
- **No reversal exists** for `destroyCluster` or `deleteClusterForce`. Those tear down cloud
  infrastructure and are one-way. Never call them as part of a remediation without saying so
  first, in those words.

## Do not

- Do not apply a rollback plan without a human reading the destroy/replace list.
- Do not assert a rollback time window. Facets publishes none.
- Do not roll back a resource in an environment where another release is in flight — you will
  get a 409, and re-reading state is the correct response, not a retry.
