---
type: Example
title: manage-a-repo — a GitHub repository as a Kubernetes resource
description: End-to-end walkthrough — install the chart, wait for the Repo RestDefinition to go Ready, wire a GitHub PAT via a Secret and a RepoConfiguration, then create, update and delete a repository as a Repo custom resource.
resource: oci://ghcr.io/krateo-blueprints/charts/github-provider-kog
tags: [example, kog, repo, github, authentication]
timestamp: 2026-08-11T00:00:00Z
---

# manage-a-repo

A complete run of the three-object pattern every kind in this provider follows: a
`Secret` (the GitHub PAT), a `*Configuration` (references the Secret), and the resource
(references the Configuration). Here the resource is a `Repo`.

The manifests referenced below are in this directory:

- `secret.yaml` — the `gh-token` Secret placeholder.
- `repoconfiguration.yaml` — a `RepoConfiguration` pointing at that Secret.
- `repo.yaml` — a `Repo` custom resource.

## Preconditions

- The [Krateo OASGen Provider](https://github.com/krateo-platformops/oasgen-provider) is
  installed (see [usage](../../docs/usage.md)).
- The chart is installed with at least the `repo` RestDefinition enabled (default):

  ```sh
  helm install github-provider krateo/github-provider-kog --namespace krateo-system
  ```

## 1. Wait for the Repo RestDefinition to be Ready

The controller must exist before you create a `Repo`:

```sh
kubectl wait restdefinitions.ogen.krateo.io github-provider-repo \
  --for condition=Ready=True --namespace krateo-system --timeout=300s
```

## 2. Provide the GitHub PAT

Edit `secret.yaml` and replace `<PAT>` with a GitHub Personal Access Token that can
create repositories in your org, then apply it:

```sh
kubectl apply -f secret.yaml
kubectl apply -f repoconfiguration.yaml
```

## 3. Create the repository

Edit `repo.yaml` — set `spec.org` to your GitHub organization — and apply it:

```sh
kubectl apply -f repo.yaml
```

Watch it reconcile. The `krateo.io/connector-verbose: "true"` annotation surfaces the
exact GitHub API calls in the controller logs:

```sh
kubectl get repo.github.ogen.krateo.io test-repo -o wide
kubectl describe repo.github.ogen.krateo.io test-repo
```

Once ready, `status` carries the repository `name`, `id` and `html_url`.

## 4. Update it

Change a mutable field — e.g. `spec.description` — and re-apply. The controller PATCHes
the repository on GitHub:

```sh
kubectl apply -f repo.yaml
```

## 5. Delete it

Deleting the CR deletes the repository on GitHub:

```sh
kubectl delete -f repo.yaml
```

## The same pattern for every kind

Swap `Repo`/`RepoConfiguration` for any other kind — `Collaborator`/`CollaboratorConfiguration`,
`TeamRepo`/`TeamRepoConfiguration`, and so on. Ready-to-apply samples for all kinds
ship under `chart/samples/`. See [api](../../docs/api.md) for the full kind list and
their verbs.
