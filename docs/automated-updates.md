# Automated workload updates

This repo keeps workloads up to date automatically, with two policies.

## Core workloads (`src/core/**`)

[Renovate](https://docs.renovatebot.com/) watches the Helm chart `targetRevision` in every
`src/core/*/application.yaml`. When a newer chart version is available it opens a **PR** (grouped as
"core charts"). A human reviews and merges — nothing auto-merges for core.

## Application workloads (`src/workloads/**`)

> **Status: dormant.** No application workload currently defines a `dev` overlay, so none is
> auto-updated, and Renovate is configured to leave `prod/` alone. The machinery below activates per
> workload as soon as that workload adds a `dev/` overlay following the convention at the end of this
> doc. `portfolio` (the only workload today) stays manually versioned.

The intended flow for a workload that has both `dev/` and `prod/` overlays (layered on a shared
`base/`):

1. **Renovate auto-updates `dev`.** New chart versions (`dev/application.yaml`) and container image
   tags (`dev/values.yaml`) are merged to `main` automatically.
2. **ArgoCD deploys dev and reports health.** When the workload's dev Application reaches `Healthy` +
   `Synced` on the new version, ArgoCD Notifications (`argocd-notifications-cm`) fires a GitHub
   `repository_dispatch` (`workload-dev-deployed`) carrying the workload name.
3. **Promotion PR for prod.** That dispatch triggers
   [`promote-workload-prod.yml`](../.github/workflows/promote-workload-prod.yml), which copies the
   dev chart version + image tags into that workload's `prod/` and opens a PR. It only opens/updates a
   PR when dev and prod actually differ, so repeated dispatches are harmless. Renovate ignores `prod/`
   entirely — this workflow is the sole owner of prod version bumps.

## Components

- [`renovate.json`](../renovate.json) — Renovate config (run by the Mend Renovate GitHub App).
- [`argocd-notifications-cm.yaml`](../src/core/argocd/manifests/argocd-notifications-cm.yaml) — the
  health gate: `on-deployed` trigger + GitHub webhook service.
- [`argocd-notifications-secret-externalsecret.yaml`](../src/core/argocd/manifests/argocd-notifications-secret-externalsecret.yaml)
  — GitHub token (from Infisical) for the webhook.
- [`promote-workload-prod.yml`](../.github/workflows/promote-workload-prod.yml) — the promotion PR.

## One-time setup

- Enable the **Mend Renovate GitHub App** on this repo and merge its onboarding PR.
- The promotion flow only needs the GitHub token once a workload actually adds a dev overlay: add the
  Infisical secret `argocd-github-dispatch-token` (prod / `/cluster`) — a GitHub PAT allowed to create
  repository_dispatch events on this repo (fine-grained: `Contents: Read and write`; or a classic PAT
  with the `repo` scope).

## Dev-overlay convention (required to activate the promotion flow for a workload)

The application-workload flow is **dormant until a `dev/` overlay exists** for that workload. Create
one mirroring `prod/`. For a workload named `<workload>`:

### `src/workloads/<workload>/dev/application.yaml`

Same multi-source shape as the prod Application, but:

- `metadata.labels.app: <workload>` (the promotion workflow reads this from the dispatch payload)
- `metadata.labels.env: dev`
- `spec.destination.namespace: <workload>-dev` (and add that namespace to the workload's `project.yaml`)
- value files: `base/values.yaml` + `dev/values.yaml`
- manifests path: `src/workloads/<workload>/dev/manifests`
- **subscribe to the health gate** via annotation:
  ```yaml
  metadata:
    annotations:
      notifications.argoproj.io/subscribe.on-deployed.github: ""
  ```

### `src/workloads/<workload>/dev/values.yaml`

Dev-specific overrides (hostname, replicas, etc.) plus the image tags. **Each tag must carry a
Renovate annotation comment** so the custom manager (which can't link the split repo/tag) tracks it:

```yaml
frontend:
  image:
    # renovate: datasource=docker depName=ghcr.io/<owner>/<workload>-frontend
    tag: "1.0.0"
backend:
  image:
    # renovate: datasource=docker depName=ghcr.io/<owner>/<workload>-backend
    tag: "1.0.0"
```

### Register it

Add `workloads/<workload>/dev/application.yaml` to [`src/kustomization.yaml`](../src/kustomization.yaml).
