---
name: Manage Facets project variables and secrets
description: Read, add and update project-level variables and secrets in a Facets blueprint, and set environment-specific values - including the gap where the REST API has no operation to list them.
api: openapi/facets-control-plane-openapi.yml
operations:
  - getStack
  - addVariable
  - addVariablesBulk
  - updateVariable
  - deleteVariables
  - getVariableAcrossEnvironments
  - getAllVariableUsages
  - getVariableUsages
  - getVariableCounts
generated: '2026-09-07'
method: generated
source: Grounded in operationIds verified verbatim in openapi/_original/facets-control-plane-openapi.json, plus https://www.facets.cloud/docs/features-and-guides/secrets-and-variables.
---

# Manage Facets project variables and secrets

Variables and secrets are declared at the **project (stack)** level and can be given
different values per **environment (cluster)**.

## The gap you need to know about first

**The REST API has no operation that lists a project's variables.** There is `POST`, `PUT`
and `DELETE` on `/cc-ui/v1/stacks/{stackName}/variables` and on
`/cc-ui/v1/designer/{stackName}/variables`, but no `GET`. What you can read is:

- `getVariableAcrossEnvironments` —
  `GET /cc-ui/v1/stacks/{stackName}/variables/{variableName}/environments` — requires you to
  already know the name;
- `getAllVariableUsages` — `GET /cc-ui/v1/designer/{stackName}/variables/usages` — which
  resources reference variables;
- `getVariableCounts` — `GET /cc-ui/v1/clusters/{clusterId}/variable-counts`.

If you need the full list, use `raptor get variables -p PROJECT`, or the
`get_secrets_and_vars` tool on the `facets-cp-mcp-server` MCP server, both of which compose
it. Do not invent a `GET` endpoint.

## Handling secrets

`Variables.secret` is a boolean on the same schema as an ordinary variable. When it is true:

- **Never echo the value.** Not into a log, not into a chat transcript, not into a commit
  message, not back to the user as confirmation.
- Confirm a write by name and by `status`, never by value.
- Facets also supports an external secrets backend
  (`/docs/features-and-guides/secrets-and-variables/secrets-backend-integration`); if the
  project uses one, the value in Facets may be a reference, not the secret itself.

## Steps

1. **Confirm the project and how it stores overrides.** `getStack` —
   `GET /cc-ui/v1/stacks/{stackName}`. If `gitOpsEnabled` or `gitOverridesEnabled` is true,
   configuration is expected to flow through Git; writing through the API may be overwritten
   by the next sync. Say so before writing.

2. **Check what a variable is used by before changing it.** `getVariableUsages` —
   `GET /cc-ui/v1/designer/{stackName}/variables/{variableName}/usages`. A variable
   referenced by several resources changes all of them at the next release.

3. **See its per-environment values.** `getVariableAcrossEnvironments` —
   `GET /cc-ui/v1/stacks/{stackName}/variables/{variableName}/environments`.

4. **Write.** `addVariable` — `POST /cc-ui/v1/stacks/{stackName}/variables`, or
   `addVariablesBulk` — `.../variables/bulk` for several at once, or `updateVariable` —
   `PUT /cc-ui/v1/stacks/{stackName}/variables`. The `Variables` body carries `description`,
   `global`, `secret`, `status` and `value`.

5. **Verify.** Re-read with `getVariableAcrossEnvironments`. This is not optional politeness:
   the API has no idempotency mechanism, so a write that timed out must be confirmed by
   reading, never by repeating.

6. **A variable change is not live until a release runs.** Changing a value updates the
   blueprint. Hand off to `facets-register-artifact-and-release.md` to roll it out, and say
   plainly that a release is required.

## Error handling

- **409 Conflict** — the name already exists (on add) or the record moved under you (on
  update). Re-read, do not retry.
- **403** — role lacks permission on secrets specifically; Facets separates secret
  management in its RBAC model.
- **404** — the variable name is right but the *stack* name is wrong. Remember the REST API
  says `stack` where the product says `project`.
