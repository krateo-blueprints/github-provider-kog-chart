---
type: Configuration
title: github-provider-kog — configuration
description: The whole values.yaml surface — the per-kind restdefinitions toggles, the proxy plugin Deployment (image, service, probes, autoscaling, scheduling), and how the plugin is auto-enabled.
resource: oci://ghcr.io/krateo-blueprints/charts/github-provider-kog
tags: [kog, values, restdefinition, plugin, helm]
timestamp: 2026-08-11T00:00:00Z
---

# Configuration

Everything is `chart/values.yaml`. The values split into two groups: the per-kind
`restdefinitions` toggles that decide which controllers get generated, and the
Deployment settings for the proxy plugin that fronts the API-inconsistent kinds.

## Which kinds to install (`restdefinitions.*`)

Each key gates the matching `RestDefinition` (and its OAS ConfigMap). All default to
`enabled: true`.

| key | default | effect |
|---|---|---|
| `restdefinitions.collaborator.enabled` | `true` | `Collaborator` RestDefinition (routed through the proxy plugin). |
| `restdefinitions.repo.enabled` | `true` | `Repo` RestDefinition (direct to `api.github.com`). |
| `restdefinitions.teamrepo.enabled` | `true` | `TeamRepo` RestDefinition (routed through the proxy plugin). |
| `restdefinitions.workflow.enabled` | `true` | `Workflow` RestDefinition (create-only, direct). |
| `restdefinitions.runnergroup.enabled` | `true` | `RunnerGroup` RestDefinition (direct). |
| `restdefinitions.gitref.enabled` | `true` | `GitRef` RestDefinition (direct). |
| `restdefinitions.repocontent.enabled` | `true` | `RepoContent` RestDefinition (direct, create-only). |
| `restdefinitions.pullrequest.enabled` | `true` | `PullRequest` RestDefinition (direct). |

Disabling a kind means its CRD and controller are never generated — use this to keep
the controller footprint to only the resources you manage.

## The proxy plugin

The [github-rest-dynamic-controller-plugin](https://github.com/krateoplatformops/github-rest-dynamic-controller-plugin)
Deployment + Service is rendered **only** when a kind that needs it is enabled — the
`github-provider-kog.plugin.enabled` helper (`_helpers.tpl`) returns `true` iff
`restdefinitions.collaborator.enabled` or `restdefinitions.teamrepo.enabled` is set.
When both are off, none of the values below have any effect because the workload is
not rendered.

### Image

| key | default | effect |
|---|---|---|
| `image.repository` | `ghcr.io/krateoplatformops/github-rest-dynamic-controller-plugin` | the plugin image. |
| `image.tag` | `""` | empty resolves to the chart `appVersion` (`0.0.3`). Set only to test another build. |
| `image.pullPolicy` | `IfNotPresent` | container image pull policy. |
| `imagePullSecrets` | `[]` | pull secrets for a private registry. |

### Naming, service account, workload

| key | default | effect |
|---|---|---|
| `replicaCount` | `1` | plugin replicas (ignored when `autoscaling.enabled`). |
| `nameOverride` / `fullnameOverride` | `""` | override the generated names. |
| `plugin.suffix` | `plugin` | suffix appended to the plugin Deployment and Service name. |
| `serviceAccount.create` | `false` | create a ServiceAccount for the plugin. |
| `serviceAccount.automount` | `true` | mount the SA token. |
| `serviceAccount.annotations` | `{}` | annotations on the created SA. |
| `serviceAccount.name` | `""` | SA name; defaults to `default` when `create=false`, else the fullname. |
| `podAnnotations` / `podLabels` | `{}` | extra pod metadata. |
| `podSecurityContext` / `securityContext` | `{}` | pod- and container-level security context. |

### Service and networking

| key | default | effect |
|---|---|---|
| `service.type` | `ClusterIP` | the plugin Service type. |
| `service.port` | `8080` | the plugin Service port (also the container port and the port the internal `webServiceUrl` helper builds). |
| `ingress.enabled` | `false` | expose the plugin via an Ingress (rarely needed — controllers reach it in-cluster). |
| `ingress.className` / `.annotations` / `.hosts` / `.tls` | see values | standard Ingress fields, applied only when enabled. |

### Health, scaling, scheduling, resources

| key | default | effect |
|---|---|---|
| `livenessProbe` | `GET /healthz:8080` | plugin liveness probe. |
| `readinessProbe` | `GET /readyz:8080` | plugin readiness probe. |
| `autoscaling.enabled` | `false` | render an HPA for the plugin. |
| `autoscaling.minReplicas` / `.maxReplicas` | `1` / `100` | HPA bounds. |
| `autoscaling.targetCPUUtilizationPercentage` | `80` | HPA CPU target. |
| `resources` | `{}` | plugin container requests/limits (unset by default). |
| `nodeSelector` / `tolerations` / `affinity` | `{}` / `[]` / `{}` | scheduling constraints. |
| `volumes` / `volumeMounts` | `[]` | extra volumes and mounts on the plugin pod. |
| `env` | `[]` | extra environment variables on the plugin container. |

## Authentication is not a chart value

The GitHub PAT and the `*Configuration` objects are **not** configured through
`values.yaml`. They are applied as separate cluster resources so credentials never live
in a Helm release. See [usage](./usage.md) and [api](./api.md) for the Secret +
Configuration contract.
