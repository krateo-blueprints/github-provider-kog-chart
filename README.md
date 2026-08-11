# GitHub Provider KOG Helm Chart

A Krateo **KOG** (Krateo Operator Generator) Helm chart that turns the GitHub REST API
into Kubernetes-native Custom Resources. It installs a set of `RestDefinition` resources
consumed by the [Krateo OASGen Provider](https://github.com/krateo-platformops/oasgen-provider),
which generates the CRDs and controllers that reconcile GitHub resources — repositories,
collaborators, team access, runner groups, workflow runs, branch refs, file commits and
pull requests — in the `github.ogen.krateo.io` API group.

## What is this

This chart carries no compiled controller. For each GitHub resource kind it declares a
`RestDefinition` (`ogen.krateo.io/v1alpha1`) that points OASGen Provider at a curated
slice of the [GitHub REST API OpenAPI spec](https://github.com/github/rest-api-description).
OASGen Provider then generates a CRD in `github.ogen.krateo.io` and deploys a dedicated
controller per kind. Two kinds (`Collaborator`, `TeamRepo`) are routed through a proxy —
the [github-rest-dynamic-controller-plugin](https://github.com/krateo-platformops/github-rest-dynamic-controller-plugin) —
that smooths over GitHub REST inconsistencies; the chart deploys it only when one of
those kinds is enabled.

Supported kinds: `Repo`, `Collaborator`, `TeamRepo`, `Workflow`, `RunnerGroup`,
`GitRef`, `RepoContent`, `PullRequest`. See [docs/overview.md](./docs/overview.md) for
the architecture and [docs/api.md](./docs/api.md) for each kind's verbs.

## Install

Prerequisite: the [Krateo OASGen Provider](https://github.com/krateo-platformops/oasgen-provider)
must already be installed. Then:

```sh
helm repo add krateo https://charts.krateo.io
helm repo update krateo
helm install github-provider krateo/github-provider-kog --namespace krateo-system
```

Installing the chart declares the RestDefinitions; OASGen Provider then generates the
CRDs and controllers, which takes a few minutes. Wait for a RestDefinition to be Ready
before creating instances of its kind:

```sh
kubectl wait restdefinitions.ogen.krateo.io github-provider-repo \
  --for condition=Ready=True --namespace krateo-system --timeout=300s
```

Full walkthrough in [docs/usage.md](./docs/usage.md).

## Configure

Every kind is independently toggleable in `values.yaml` via `restdefinitions.<kind>.enabled`
(all default `true`), so you install only the controllers you need. Disabling both
`collaborator` and `teamrepo` also drops the proxy plugin Deployment. The plugin's
Deployment settings (image, service, probes, autoscaling, scheduling) are the rest of
the value surface.

```yaml
restdefinitions:
  repo:
    enabled: true
  collaborator:
    enabled: false
  teamrepo:
    enabled: false
```

Authentication is **not** a chart value: apply a Kubernetes `Secret` with a GitHub PAT
and a `*Configuration` object that references it, then point each resource at the
Configuration via `spec.configurationRef`. The full value surface is in
[docs/configuration.md](./docs/configuration.md); the auth contract is in
[docs/api.md](./docs/api.md).

## Examples

- [examples/manage-a-repo](./examples/manage-a-repo/README.md) — end-to-end: install,
  wait for the `Repo` RestDefinition, wire a PAT via a Secret and a `RepoConfiguration`,
  then create/update/delete a repository as a `Repo` custom resource.

Ready-to-apply CRs for every kind also ship under `chart/samples/` (resources) and
`chart/samples/configs/` (the matching `*Configuration` objects). More in
[docs/examples.md](./docs/examples.md).

## Docs

- [docs/index.md](./docs/index.md) — the doc bundle map.
- [docs/overview.md](./docs/overview.md) — KOG/OASGen architecture, the eight kinds,
  the proxy plugin.
- [docs/usage.md](./docs/usage.md) — prerequisites, install, auth, first resource.
- [docs/configuration.md](./docs/configuration.md) — the whole `values.yaml` surface.
- [docs/api.md](./docs/api.md) — the generated CRDs, verbs, and authentication.
- [docs/examples.md](./docs/examples.md) — the runnable examples.
- [docs/release.md](./docs/release.md) — how a tag ships the chart.
- [docs/log.md](./docs/log.md) — curated history.
- [docs/troubleshooting.md](./docs/troubleshooting.md) — per-kind common issues.
- [docs/llms.txt](./docs/llms.txt) — the doc index for LLM consumers.

## Develop & release

The chart lives under `chart/`. `templates/rd-<kind>.yaml` declares each RestDefinition,
`templates/configmap-<kind>.yaml` wraps its OpenAPI fragment from `assets/<kind>.yaml`,
and `templates/deployment.yaml`/`service.yaml` render the proxy plugin (gated by the
`github-provider-kog.plugin.enabled` helper in `_helpers.tpl`).

Releases are cut by pushing a plain-semver tag (`X.Y.Z`, no `v` prefix) that **matches**
`chart/Chart.yaml` `version`; `.github/workflows/release-tag.yaml` lint-tests,
packages and pushes the chart to `oci://ghcr.io/krateo-blueprints/charts/github-provider-kog`.
`.github/workflows/lint.yaml` runs the shared Krateo docs-standard linter on every PR.
Details in [docs/release.md](./docs/release.md).
