---
type: ExampleIndex
title: github-provider-kog — examples
description: Index of the runnable examples under examples/.
resource: oci://ghcr.io/krateo-blueprints/charts/github-provider-kog
tags: [examples, kog, github]
timestamp: 2026-08-11T00:00:00Z
---

# Examples

- [examples/manage-a-repo](../examples/manage-a-repo/README.md) — end-to-end: install
  the chart, wait for the `Repo` RestDefinition to go Ready, wire a GitHub PAT via a
  `Secret` and a `RepoConfiguration`, then create, update and delete a repository as a
  `Repo` custom resource. It doubles as the reference for the Secret + Configuration +
  resource three-object pattern that every kind in this provider follows.

Ready-to-apply CRs for every supported kind also ship in the chart itself under
`chart/samples/` (resources) and `chart/samples/configs/` (the matching
`*Configuration` objects).
