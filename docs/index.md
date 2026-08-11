---
type: Component
title: github-provider-kog — index
description: The map of the github-provider-kog doc bundle — a Krateo KOG (Krateo Operator Generator) Helm chart that turns the GitHub REST API into Kubernetes-native Custom Resources via the OASGen Provider.
resource: oci://ghcr.io/krateo-blueprints/charts/github-provider-kog
tags: [kog, oasgen-provider, github, restdefinition, helm-chart]
timestamp: 2026-08-11T00:00:00Z
---

# github-provider-kog

`github-provider-kog` is a Krateo **KOG** (Krateo Operator Generator) Helm chart. It
does not carry hand-written controllers: it installs a set of `RestDefinition`
resources that hand the [GitHub REST API OpenAPI specification](https://github.com/github/rest-api-description)
to the [Krateo OASGen Provider](https://github.com/krateoplatformops/oasgen-provider),
which in turn generates the CRDs and spins up the controllers that reconcile GitHub
resources. Once installed, you manage GitHub repositories, collaborators, team access,
runner groups, workflow runs, branch references, file commits and pull requests as
Kubernetes objects in the `github.ogen.krateo.io` API group.

## The bundle (start here)

- [overview](./overview.md) — the KOG/OASGen architecture, what the chart installs,
  the eight supported kinds, and why a proxy plugin sits in front of some of them.
- [usage](./usage.md) — prerequisites, install, wait for the RestDefinitions to go
  Ready, authenticate, and create your first resource.
- [configuration](./configuration.md) — the whole `values.yaml` surface: the
  per-kind `restdefinitions` toggles, the plugin Deployment, image, service, probes.
- [api](./api.md) — the generated CRDs (the eight resource kinds + their
  `*Configuration` companions), the verbs each RestDefinition wires, and the
  authentication contract.
- [examples](./examples.md) — the runnable example under `examples/`.
- [release](./release.md) — how a tag ships the chart to GHCR OCI.
- [log](./log.md) — curated history.
- [llms.txt](./llms.txt) — the doc index for LLM consumers.

## Layout

- `chart/` — the Helm chart (`type: application`, name `github-provider-kog`).
  - `templates/rd-<kind>.yaml` — the eight `RestDefinition` resources, each gated by a
    `restdefinitions.<kind>.enabled` value.
  - `templates/configmap-<kind>.yaml` — the OpenAPI spec for each kind, wrapped as a
    ConfigMap and referenced by the matching RestDefinition's `oasPath`.
  - `assets/<kind>.yaml` — the curated OpenAPI (OAS) fragment per kind.
  - `templates/deployment.yaml` + `templates/service.yaml` — the
    [github-rest-dynamic-controller-plugin](https://github.com/krateoplatformops/github-rest-dynamic-controller-plugin),
    a proxy that smooths over GitHub REST inconsistencies; rendered **only** when a
    kind that needs it (`collaborator`, `teamrepo`) is enabled.
  - `samples/` — ready-to-apply example CRs; `samples/configs/` — the matching
    `*Configuration` resources.
- `docs/crds/` — the generated CRDs, checked in for reference only (OASGen Provider
  applies them at runtime, not the chart).
- `docs/troubleshooting.md` — common issues per resource kind.

This is a single-chart repo (`chart/`), so this index is typed `Component`.
