---
type: Architecture
title: github-provider-kog — overview
description: How the chart works — the KOG/OASGen Provider architecture, the RestDefinition-to-controller flow, the eight GitHub resource kinds, and the proxy plugin that fronts the API-inconsistent ones.
resource: oci://ghcr.io/krateo-blueprints/charts/github-provider-kog
tags: [kog, oasgen-provider, restdefinition, github-plugin, architecture]
timestamp: 2026-08-11T00:00:00Z
---

# Overview

`github-provider-kog` is a **KOG** chart: it ships no compiled controller of its own.
Instead it declares, for each GitHub resource kind, a `RestDefinition`
(`ogen.krateo.io/v1alpha1`) that points the [Krateo OASGen Provider](https://github.com/krateo-platformops/oasgen-provider)
at a curated slice of the GitHub REST API's OpenAPI specification. OASGen Provider
reads that spec, generates a CRD in the `github.ogen.krateo.io` group, and deploys a
dedicated controller that reconciles instances of that CRD against the live GitHub API.

## The install flow

One `helm install` lays down, per enabled kind:

1. a **ConfigMap** (`templates/configmap-<kind>.yaml`) holding the OpenAPI (OAS)
   fragment from `assets/<kind>.yaml`;
2. a **RestDefinition** (`templates/rd-<kind>.yaml`) whose `spec.oasPath` is a
   `configmap://<namespace>/<release>-<kind>/<kind>.yaml` URL pointing at that ConfigMap,
   plus a `resource` block naming the generated `kind`, its `additionalStatusFields`,
   and a `verbsDescription` mapping each verb (create/get/update/delete) to an HTTP
   method + path on the GitHub API.

OASGen Provider then, asynchronously:

- generates the CRD (e.g. `repoes.github.ogen.krateo.io`) and a companion
  `*Configuration` CRD that carries authentication;
- deploys a controller Deployment (e.g. `<release>-repo-controller`);
- marks the RestDefinition `Ready=True` once the CRD is served and the controller is
  up. This takes a few minutes after `helm install` returns — the chart install
  completing is **not** the same as the resources being usable ([usage](./usage.md)).

Because each kind is an independent RestDefinition gated by
`restdefinitions.<kind>.enabled`, you install only the controllers you need.

## The eight resource kinds

| Kind | Group | Manages | Notable verbs |
|---|---|---|---|
| `Repo` | `github.ogen.krateo.io` | GitHub repositories | create/get/update/delete |
| `Collaborator` | `github.ogen.krateo.io` | repo collaborators + external invitations | create/get/update/delete (via plugin) |
| `TeamRepo` | `github.ogen.krateo.io` | team access to a repo | create/get/update/delete (via plugin) |
| `Workflow` | `github.ogen.krateo.io` | `workflow_dispatch` runs | create only |
| `RunnerGroup` | `github.ogen.krateo.io` | Actions runner groups | create/get/update/delete |
| `GitRef` | `github.ogen.krateo.io` | branch references | create/get/delete |
| `RepoContent` | `github.ogen.krateo.io` | one committed file per CR | create/delete (v1 create-only) |
| `PullRequest` | `github.ogen.krateo.io` | pull requests | create/get/update (no delete) |

The full verb/path table and the field-mapping rationale are in [api](./api.md).

## The proxy plugin — fronted only where the API demands it

Two kinds — `Collaborator` and `TeamRepo` — do not talk to `api.github.com` directly.
Their RestDefinition paths (e.g. `/repository/{owner}/{repo}/collaborators/{username}/permission`)
target the [github-rest-dynamic-controller-plugin](https://github.com/krateo-platformops/github-rest-dynamic-controller-plugin),
a small proxy that resolves inconsistencies in the GitHub REST API (for example, the
permission-read shape) that the generated controllers cannot express directly.

The chart deploys that plugin as a **Deployment + Service**
(`templates/deployment.yaml`, `templates/service.yaml`) — but only when a kind that
needs it is enabled. The gate is the `github-provider-kog.plugin.enabled` helper
(`_helpers.tpl`), which returns `true` only if `collaborator` or `teamrepo` is enabled.
Disable both and the plugin workload is not rendered at all; the remaining kinds call
`api.github.com` directly. The plugin image tag is pinned by the chart's `appVersion`.

## Git-write kinds and the GitOps chain

`GitRef`, `RepoContent` and `PullRequest` were added to support a GitOps publish flow —
create a branch, commit files onto it, then open a pull request — driven entirely from
Kubernetes CRs. They call the GitHub API directly (no plugin) and each encodes a small
GitHub-API quirk in its RestDefinition:

- **`GitRef`** — the create body wants the *fully qualified* ref
  (`refs/heads/<branch>`) while get/delete want `heads/<branch>`; the `heads/` prefix
  is baked into the path templates, so the CR carries both `ref` (for create) and
  `branch` (for get/delete). Adopting an existing branch surfaces its head sha in
  `status.object.sha`.
- **`RepoContent`** — one CR commits one Base64-encoded file; the blob sha from the
  create response (`status.content.sha`) is what the delete verb uses. v1 is
  create-only: a get+update pair would perpetually re-commit because GitHub returns
  file `content` newline-wrapped and it never equals the client-encoded spec value.
- **`PullRequest`** — `head_ref`/`base_ref` on the spec map to the API's `head`/`base`
  body fields (GitHub returns those as objects on reads, so distinct names avoid a
  type mismatch); `state: closed` closes the PR, and since GitHub PRs cannot be API-deleted,
  deleting the CR just releases it.

## Authentication

Every resource references a companion `*Configuration` object (e.g. `RepoConfiguration`)
via `spec.configurationRef`. That Configuration holds a bearer-token reference to a
Kubernetes `Secret` containing a GitHub Personal Access Token. One Configuration can be
shared by many resources of the same kind, and it may live in a different namespace.
The authentication contract is detailed in [api](./api.md) and [usage](./usage.md).
