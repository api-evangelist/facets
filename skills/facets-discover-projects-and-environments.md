---
name: Discover Facets projects and environments
description: Orient inside a Facets control plane - confirm reachability, list projects (stacks), list their environments (clusters), and read the current state of one - before taking any action.
api: openapi/facets-control-plane-openapi.yml
operations:
  - healthCheck
  - getStacks
  - getStack
  - getClusters
  - getClusterCommon
  - getClusterState
  - getLatestRelease
generated: '2026-09-07'
method: generated
source: Grounded in operationIds verified verbatim in openapi/_original/facets-control-plane-openapi.json and in https://www.facets.cloud/docs/api.
---

# Discover Facets projects and environments

Run this before any other Facets skill. It establishes which control plane you are on, what
exists on it, and what state it is in.

## Before you start

**Base URL.** Every customer has their own control plane host:
`https://<account-id>.console.facets.cloud`. There is no shared api.facets.cloud. Get the
host from `CONTROL_PLANE_URL`, or from `~/.facets/credentials` if raptor or praxis has been
logged in on this machine. Do not guess it.

**Auth.** HTTP Basic. Username is the operator's sign-in email; password is a personal
access token from Account Settings -> Personal Token (`<control-plane-url>/v2/home#personal-access-tokens`).
In CI, read `FACETS_USERNAME` and `FACETS_TOKEN`. Never put the token in a URL or a log line.

**Vocabulary — this is the single most common source of a wrong call.** The REST API and the
product use different words for the same object:

| REST says | Product, CLI and MCP say |
|---|---|
| `stack` | project |
| `cluster` | environment |
| `DeploymentLog` | release |
| `artifact` | build |
| `artifactCI` | artifact |

When a human asks about "the staging environment of the payments project", you are looking
for the cluster named `staging` under the stack named `payments`.

## Steps

1. **Confirm the control plane is up.** `healthCheck` — `GET /public/v1/health`. This is one
   of the ten operations that need no credentials, so it also tells you whether the host is
   right before you spend a token on it.

2. **List the projects.** `getStacks` — `GET /cc-ui/v1/stacks/`. Returns an unbounded array
   of `Stack`; there is no pagination on this operation, so on a large tenant expect a large
   body. The join key for everything downstream is `Stack.name`, not `Stack.id`.

3. **Read one project.** `getStack` — `GET /cc-ui/v1/stacks/{stackName}`. Fields worth
   reading before you act: `pauseReleases` (releases are frozen), `gitOpsEnabled` and
   `gitOverridesEnabled` (overrides live in Git, so writing them through the API may be the
   wrong path), `primaryCloud`, `allowedClouds`, `childStacks`.

4. **List its environments.** `getClusters` — `GET /cc-ui/v1/stacks/{stackName}/clusters`.
   Each is an `AbstractCluster`. Check `isEphemeral` (short-lived environment, may be torn
   down on a schedule), `baseClusterId` (this environment inherits from another),
   `pauseReleases`, `requireSignOff`, and `deleted`.

5. **Read one environment.** `getClusterCommon` — `GET /cc-ui/v1/clusters/{clusterId}`, and
   `getClusterState` — `GET /cc-ui/v1/clusters/{clusterId}/deployments/state` for its
   deployment state.

6. **Find its last good release.** `getLatestRelease` —
   `GET /cc-ui/v1/clusters/{clusterId}/deployments/latest-successful-release`. Note the
   returned `DeploymentLog.id`; you need it if you later have to roll back, and note whether
   it carries `deploymentContextFilePath` — a release without one cannot be a rollback
   target.

## Error handling

Every operation declares the same six failures: 400, 403, 404, 405, 409, 500, all with a
`{code, message}` body (`ErrorDetails`). Specifics:

- **403** — the credential is valid but the role lacks the permission. If you are running as
  an agent, remember Facets caps AI permissions at or below the acting user's own, so you can
  get a 403 on something that user could do by hand.
- **404** — check the vocabulary table above before concluding the object is missing.
- **401** is *not* declared in the spec but the API does return it; treat it as a bad or
  expired token.

Do not build retry logic around 429 — Facets publishes no rate limits and declares no 429.

## What this skill must not do

Read-only. If the answer requires creating, releasing, destroying or overriding anything,
stop and hand off to the relevant write skill, and get the human's confirmation first.
