---
type: Runbook
title: github-provider-kog — release
description: How a release ships — a plain-semver git tag matching Chart.yaml version drives the release-chart workflow to package the chart and push it to the krateo-blueprints charts OCI registry on GHCR.
resource: oci://ghcr.io/krateo-blueprints/charts/github-provider-kog
tags: [release, oci, ghcr, helm]
timestamp: 2026-08-11T00:00:00Z
---

# Release

A single plain-semver tag (`X.Y.Z`, **no** `v` prefix) publishes the chart. The tag
push triggers `.github/workflows/release-chart.yaml`.

## What a tag ships

`release-chart.yaml`, on a tag matching `[0-9]+.[0-9]+.[0-9]+` (or a manual
`workflow_dispatch`):

1. `helm lint chart` and a `helm template` smoke test render the chart.
2. A guard asserts the git tag **equals** `chart/Chart.yaml` `version` — a mismatch
   fails the release (that is why `Chart.yaml` carries a concrete version, not a
   placeholder).
3. `helm package chart` then `helm push` publishes to
   `oci://ghcr.io/<owner>/charts` — i.e.
   `oci://ghcr.io/krateo-blueprints/charts/github-provider-kog`. The owner namespace is
   derived from the repository owner (a hardcoded org would 403 the moment the repo
   moved), and the push retries to ride out GHCR first-push flakiness.

This is the same OCI convention the sibling KOG charts (`hetzner-object-storage-kog`,
`krateo-oxide-operator-kog`) use; the Krateo Installer's `CompositionDefinition`
consumes the published artifact via `spec.chart.url`.

## Steps

```console
$ git tag X.Y.Z && git push origin X.Y.Z
```

Then verify the artifact exists:

```console
$ helm show chart oci://ghcr.io/krateo-blueprints/charts/github-provider-kog --version X.Y.Z | head -3
```

Bump `chart/Chart.yaml` `version` **before** tagging, or the tag/version guard fails.
When updating the proxy plugin, bump `appVersion` (it pins the
`github-rest-dynamic-controller-plugin` image tag).

## PR-time checks

`lint.yaml` runs the shared Krateo docs-standard linter (`lint-docs`) on every PR, so
this documentation bundle is validated before merge. The release workflow additionally
runs `helm lint` and a template smoke test at tag time.

## Version pinning downstream

Consumers install by explicit `--version`; nothing tracks a mutable tag. The chart
`version` and the git tag are always equal by construction.
