---
type: Usage
title: github-provider-kog — usage
description: How to install and use the provider — prerequisites, helm install, waiting for the RestDefinitions to go Ready, wiring a GitHub PAT via a Secret and a Configuration, and creating your first resource.
resource: oci://ghcr.io/krateo-blueprints/charts/github-provider-kog
tags: [kog, install, restdefinition, authentication, github]
timestamp: 2026-08-11T00:00:00Z
---

# Usage

## Prerequisites

The [Krateo OASGen Provider](https://github.com/krateoplatformops/oasgen-provider)
must already be installed in the cluster — this chart declares `RestDefinition`
resources that OASGen Provider consumes. Follow the
[oasgen-provider-chart README](https://github.com/krateoplatformops/oasgen-provider-chart)
to install it first.

## Install

```sh
helm repo add krateo https://charts.krateo.io
helm repo update krateo
helm install github-provider krateo/github-provider-kog --namespace krateo-system
```

Installing the chart creates a `RestDefinition` (and its OAS ConfigMap) per enabled
kind, plus the proxy plugin Deployment/Service if `collaborator` or `teamrepo` is
enabled. OASGen Provider then generates the CRDs and deploys the controllers — this
takes a few minutes and is **not** finished when `helm install` returns.

## Wait for the RestDefinitions to be Ready

A RestDefinition reaches `Ready` only once its CRD is served and its controller is up.
List them:

```sh
kubectl get restdefinitions.ogen.krateo.io -A | awk 'NR==1 || /github/'
```

```sh
NAMESPACE       NAME                           READY   AGE
krateo-system   github-provider-collaborator   False   24s
krateo-system   github-provider-repo           False   24s
krateo-system   github-provider-runnergroup    False   24s
krateo-system   github-provider-teamrepo       False   24s
krateo-system   github-provider-workflow       False   24s
```

Wait for a specific one before creating instances of its kind:

```sh
kubectl wait restdefinitions.ogen.krateo.io github-provider-repo \
  --for condition=Ready=True --namespace krateo-system --timeout=300s
```

The RestDefinition names and namespace vary with your release name and install
namespace. See [troubleshooting](./troubleshooting.md) for checking the generated CRDs
and controllers.

## Authenticate — a Secret plus a Configuration

Two objects are required to talk to the GitHub REST API.

First, a `Secret` holding a GitHub [Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-personal-access-token-classic)
with the scopes for the resources you manage:

```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: gh-token
  namespace: default
type: Opaque
stringData:
  token: <PAT>
EOF
```

Then a `*Configuration` object that references that Secret. The kind depends on the
resource you manage — a `Repo` needs a `RepoConfiguration`, a `Collaborator` needs a
`CollaboratorConfiguration`, and so on:

```sh
kubectl apply -f - <<EOF
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
EOF
```

A single Configuration can be shared by many resources of the same kind, and it may
live in a different namespace than the resources that reference it.

## Create your first resource

Reference the Configuration from the resource via `spec.configurationRef`:

```yaml
apiVersion: github.ogen.krateo.io/v1alpha1
kind: Repo
metadata:
  name: test-repo
  namespace: default
  annotations:
    krateo.io/connector-verbose: "true"
spec:
  configurationRef:
    name: my-repo-config
    namespace: default
  org: krateoplatformops-test
  name: test-repo
  description: A short description of the repository set by Krateo
  visibility: public
  has_issues: true
```

Ready-to-apply samples for every kind live under `chart/samples/` (and their matching
Configurations under `chart/samples/configs/`). See [examples](./examples.md) for a
complete end-to-end walkthrough.

## Install only the kinds you need

Every kind is independently toggleable in `values.yaml`. To run only the `Repo`
controller:

```yaml
restdefinitions:
  collaborator:
    enabled: false
  repo:
    enabled: true
  teamrepo:
    enabled: false
  workflow:
    enabled: false
  runnergroup:
    enabled: false
  gitref:
    enabled: false
  repocontent:
    enabled: false
  pullrequest:
    enabled: false
```

Disabling `collaborator` and `teamrepo` together also drops the proxy plugin
Deployment. The full value surface is in [configuration](./configuration.md).

## Verbose logging

Add the `krateo.io/connector-verbose: "true"` annotation to any resource to get
detailed operation logs from its controller — the fastest way to see the exact GitHub
API request/response when a resource will not reconcile.
