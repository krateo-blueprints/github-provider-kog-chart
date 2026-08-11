---
type: API
title: github-provider-kog — API
description: The generated CRDs — the eight GitHub resource kinds in github.ogen.krateo.io, their companion Configuration kinds, the verbs each RestDefinition wires to the GitHub API, and the authentication contract.
resource: oci://ghcr.io/krateo-blueprints/charts/github-provider-kog
tags: [kog, crd, restdefinition, github-api, authentication]
timestamp: 2026-08-11T00:00:00Z
---

# API

This chart installs no CRDs directly. It installs `RestDefinition` resources
(`ogen.krateo.io/v1alpha1`); the [OASGen Provider](https://github.com/krateo-platformops/oasgen-provider)
reads each one's OpenAPI spec and **generates** the CRDs described here, all in the
`github.ogen.krateo.io/v1alpha1` group/version. The checked-in copies under
`docs/crds/` are for reference only.

## The RestDefinition input

Each `templates/rd-<kind>.yaml` is shaped like this (`Repo` shown):

```yaml
kind: RestDefinition
apiVersion: ogen.krateo.io/v1alpha1
metadata:
  name: <release>-repo
spec:
  oasPath: configmap://<namespace>/<release>-repo/repo.yaml
  resourceGroup: github.ogen.krateo.io
  resource:
    kind: Repo
    additionalStatusFields:
      - name
      - id
      - html_url
    verbsDescription:
      - action: create
        method: POST
        path: /orgs/{org}/repos
      - action: get
        method: GET
        path: /repos/{org}/{name}
      - action: update
        method: PATCH
        path: /repos/{org}/{name}
      - action: delete
        method: DELETE
        path: /repos/{org}/{name}
```

- `oasPath` points at the ConfigMap (`templates/configmap-<kind>.yaml`) that wraps the
  OpenAPI fragment from `assets/<kind>.yaml`.
- `resource.kind` becomes the generated CRD's kind in `github.ogen.krateo.io`.
- `additionalStatusFields` selects GitHub response fields to surface under
  `status`.
- `verbsDescription` maps each lifecycle verb to an HTTP method and path.

## Generated resource kinds

All in `github.ogen.krateo.io/v1alpha1`. "via plugin" means the path targets the proxy
Deployment, not `api.github.com` directly.

| Kind | Verbs (method → path) | Notes |
|---|---|---|
| `Repo` | POST `/orgs/{org}/repos`; GET/PATCH/DELETE `/repos/{org}/{name}` | full CRUD. |
| `Collaborator` | create/update/delete `/repository/{owner}/{repo}/collaborators/{username}`; get `.../permission` (all **via plugin**) | supports external-collaborator invitations; status carries `permissions` and `message`. |
| `TeamRepo` | CRUD **via plugin** on team/repo permission | grants a team access to a repo. |
| `Workflow` | create only — `workflow_dispatch` | triggers Actions runs; no get/update/delete (a new run is a new create). |
| `RunnerGroup` | full CRUD | manages Actions runner groups. |
| `GitRef` | POST `/repos/{owner}/{repo}/git/refs`; GET/DELETE `.../git/ref[s]/heads/{branch}` | create wants fully-qualified `ref`, get/delete want `heads/{branch}`; adopting an existing branch surfaces its head in `status.object.sha`. |
| `RepoContent` | PUT/DELETE `/repos/{owner}/{repo}/contents/{path}` | one Base64 file per CR; **create-only** in v1; delete uses the blob sha from `status.content.sha`. |
| `PullRequest` | POST `/repos/{owner}/{repo}/pulls`; GET/PATCH `.../pulls/{pull_number}` | no delete (GitHub PRs cannot be API-deleted); `state: closed` closes it. |

### Field-mapping details worth knowing

Some RestDefinitions carry `requestFieldMapping` / `excludedSpecFields` to reconcile
GitHub's request/response asymmetries:

- **`PullRequest`** — spec fields `head_ref`/`base_ref` are mapped into the API body's
  `head`/`base` (GitHub returns those as objects on reads, so distinct spec names avoid
  a perpetual type mismatch). `pull_number` is server-assigned and sourced from
  `status.number` on get/update. `head`, `base`, `pull_number` are excluded from the
  generated spec.
- **`RepoContent`** — `sha` is excluded from the user-facing spec (it is
  server-derived) and injected into the delete body from `status.content.sha`. Status
  carries `content.sha`, `content.html_url`, `commit.sha`.
- **`GitRef`** — the `heads/` prefix is embedded in the get/delete path templates; the
  CR carries both `ref` (fully qualified, for create) and `branch` (plain, for
  get/delete).

## Authentication — Configuration companions

Every resource references a companion `*Configuration` object through
`spec.configurationRef`. OASGen Provider generates one Configuration kind per resource
kind:

- `RepoConfiguration`
- `CollaboratorConfiguration`
- `TeamRepoConfiguration`
- `WorkflowConfiguration`
- `RunnerGroupConfiguration`

(plus the git-write kinds' configurations where applicable). A Configuration holds a
bearer-token reference to a Kubernetes `Secret` carrying a GitHub PAT:

```yaml
apiVersion: github.ogen.krateo.io/v1alpha1
kind: RepoConfiguration
metadata:
  name: my-repo-config
  namespace: default
spec:
  authentication:
    bearer:
      tokenRef:
        name: gh-token
        namespace: default
        key: token
```

The resource then points at it:

```yaml
apiVersion: github.ogen.krateo.io/v1alpha1
kind: Repo
metadata:
  name: test-repo
spec:
  configurationRef:
    name: my-repo-config
    namespace: default
  org: krateo-platformops-test
  name: test-repo
```

A single Configuration can be referenced by many resources of the same kind, and may
live in a different namespace than the resources that use it.

## Operator annotation

`krateo.io/connector-verbose: "true"` on any resource turns on verbose logging in its
controller — see [usage](./usage.md) and [troubleshooting](./troubleshooting.md).
