---
name: Register a build and release it to an environment
description: The core Facets CI flow - register a container image or zip against a named CI integration, confirm it landed, then trigger and watch a release on one environment.
api: openapi/facets-control-plane-openapi.yml
operations:
  - getArtifactCiByName
  - registerArtifactV2
  - getArtifactsForCI
  - promoteArtifact
  - createDeployment
  - releaseV2
  - getDeployment
  - getDeploymentLogs
  - getReleaseChanges
  - abortRelease
generated: '2026-09-07'
method: generated
source: Grounded in operationIds verified verbatim in openapi/_original/facets-control-plane-openapi.json, plus https://www.facets.cloud/docs/features-and-guides/integration-and-delivery/integrating-with-ci-pipelines.
---

# Register a build and release it to an environment

This is the flow a CI pipeline runs after it has built and pushed an image. It **changes
production infrastructure.** Read the whole of "Before you act" before calling anything.

## Before you act

**Two different things are both called "artifact".** In the REST API, an `ArtifactCI` is the
*named CI integration* a resource deploys from, and an `Artifact` is *one image or zip*
registered under it. The product UI calls the first an artifact and the second a build. You
register a build against a CI integration that already exists — you do not create the
integration in this flow.

**There is no idempotency.** The Facets API has no `Idempotency-Key` header and no request
id. If `registerArtifactV2` or `releaseV2` times out, you do not know whether it landed. Do
not blind-retry a write. Re-read state first (step 3, step 7) and decide from what you see.

**Prefer V2.** `registerArtifact` (`POST /cc-ui/v1/artifacts/register`) is marked
`deprecated: true` in the spec. Use `registerArtifactV2` (`POST /cc-ui/v1/artifacts/registerV2`).

## Steps

1. **Resolve the CI integration.** `getArtifactCiByName` —
   `GET /cc-ui/v1/artifacts-ci/name/{ciName}`. Confirms the integration exists and gives you
   its `stackName` and `promotionWorkflowId`. If this 404s, the integration has not been set
   up; stop and tell the human — do not create one implicitly.

2. **Register the build.** `registerArtifactV2` — `POST /cc-ui/v1/artifacts/registerV2`. The
   `Artifact` body carries `artifactUri`, `buildId`, `tag`, `repositoryName`, `artifactory`,
   `releaseStream`, `releaseType` and an open `metadata` object. Every environment qualified
   for that release stream gets the artifact in queue.

3. **Confirm it landed.** `getArtifactsForCI` —
   `GET /cc-ui/v1/artifacts-ci/{ciName}/artifacts`. Do this even on a clean 200; it is also
   how you recover from an ambiguous timeout in step 2 without double-registering.

4. **Promote if the ladder requires it.** `promoteArtifact` —
   `POST /cc-ui/v1/artifacts/{ciId}/promote/{artifactId}` moves a build up the promotion
   ladder. Skip this if the target environment's release stream already qualifies the build.

5. **Ask the human before releasing.** A release runs Terraform against real cloud
   infrastructure. State plainly which environment (`clusterId`), which build, and what the
   release type is. Wait for confirmation.

6. **Trigger the release.** `releaseV2` —
   `PUT /cc-ui/v1/clusters/{clusterId}/deployments/releaseV2/{releaseType}`. (`release` at
   `PUT /cc-ui/v1/clusters/{clusterId}/deployments/release` and `createDeployment` at
   `POST /cc-ui/v1/clusters/{clusterId}/deployments` are the older and lower-level forms.)
   Keep the returned `DeploymentLog.id` and its `releaseTraceId` — the trace id is the only
   correlation handle this API has.

7. **Watch it.** Poll `getDeployment` —
   `GET /cc-ui/v1/clusters/{clusterId}/deployments/{deploymentId}` for `status` and
   `currentPhase`. `getDeploymentLogs` (`.../logs`) gives the phase logs and `errorLogs`;
   `getReleaseChanges` (`.../release-changes`) gives the Terraform changes that ran. Back off
   exponentially between polls — the API publishes no rate limits, which is a reason to be
   conservative, not a licence to hammer it.

8. **If it goes wrong while running:** `abortRelease` —
   `POST /cc-ui/v1/clusters/{clusterId}/deployments/{deploymentId}/abort`. This only works
   while the release is in flight. Once it has finished, the reversal path is a rollback —
   see `facets-rollback-a-release.md`.

## Error handling

- **409 Conflict** — most often a release is already running on that environment, or the
  build id is already registered. Re-read (step 3 or `getDeployments`) rather than retrying.
- **400** — if the body carries `missingPermissions[]` and `approvalUrl`, the linked GitHub
  App is short a permission; send a human to that URL. Otherwise it is a validation failure
  against the request schema.
- **403** — role lacks release authority, or the role's AI Permissions have been narrowed
  below the user's own.
- **500** — safe to retry only after re-reading state, because of the idempotency gap above.

## Do not

- Do not call `registerArtifact` (deprecated) when `registerArtifactV2` exists.
- Do not release without explicit human confirmation of the environment.
- Do not retry a write on timeout without re-reading.
