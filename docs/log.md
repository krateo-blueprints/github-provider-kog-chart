---
type: Log
title: github-provider-kog — log
description: Curated chronological history of github-provider-kog — notable changes and decisions, not a generated changelog.
resource: oci://ghcr.io/krateo-blueprints/charts/github-provider-kog
tags: [log, history]
timestamp: 2026-08-11T00:00:00Z
---

# Log

Curated history; release notes live in GitHub Releases.

## 2026-08-11 — Documentation Standard adoption

The repo adopts the Krateo Documentation Standard (OKF): the invariant docs bundle
(`index`, `overview`, `usage`, `configuration`, `api`, `examples`, `release`, `log`
plus `llms.txt`), a runnable `examples/manage-a-repo`, and the shared `lint-docs` check
wired into a new `lint.yaml`. `README.md` is rewritten to the six-section standard
shape. Part of krateo-platformops/installer#52.

## 0.3.0 — git-write kinds

Added `GitRef`, `RepoContent` and `PullRequest` to support a GitOps publish flow
(create branch → commit files → open pull request) driven from Kubernetes CRs. Each
encodes a GitHub-API quirk in its RestDefinition:

- `GitRef` carries both a fully-qualified `ref` (for create) and a plain `branch` (for
  get/delete, with `heads/` baked into the path); adopting an existing branch surfaces
  its head sha.
- `RepoContent` is create-only in v1 — GitHub returns file content newline-wrapped, so
  a get+update pair would perpetually re-commit.
- `PullRequest` maps `head_ref`/`base_ref` to the API's `head`/`base` and has no delete
  verb (GitHub PRs cannot be API-deleted).

These kinds call `api.github.com` directly, bypassing the proxy plugin.

## Earlier — the core provider

The initial provider shipped `Repo`, `Collaborator`, `TeamRepo`, `Workflow` and
`RunnerGroup` as RestDefinitions over the GitHub REST API OpenAPI spec, with the
`github-rest-dynamic-controller-plugin` fronting `Collaborator` and `TeamRepo` to
smooth over GitHub REST inconsistencies. The plugin Deployment is rendered only when a
kind that needs it is enabled.
