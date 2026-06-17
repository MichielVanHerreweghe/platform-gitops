# Automated workload updates

This repo keeps workloads up to date automatically, with two policies.

## Core workloads (`src/core/**`)

[Renovate](https://docs.renovatebot.com/) watches the Helm chart `targetRevision` in every
`src/core/*/application.yaml`. When a newer chart version is available it opens a **PR** (grouped as
"core charts"). A human reviews and merges — nothing auto-merges for core.

## Application workloads (`src/workloads/**`)

Per-environment overlays (`dev/`, `prod/`) layered on a shared `base/`. The update flow:

1. **Renovate auto-updates `dev`.** New chart versions (`dev/application.yaml`) and container image
   tags (`dev/values.yaml`) are merged to `main` automatically.
2. **ArgoCD deploys dev and reports health.** When the `*-dev` Application reaches `Healthy` +
   `Synced` on the new version, ArgoCD Notifications (`argocd-notifications-cm`) fires a GitHub
   `repository_dispatch` (`portfolio-dev-deployed`).
3. **Promotion PR for prod.** That dispatch triggers
   [`promote-portfolio-prod.yml`](../.github/workflows/promote-portfolio-prod.yml), which copies the
   dev chart version + image tags into `prod/` and opens a PR. It only opens/updates a PR when dev
   and prod actually differ, so repeated dispatches are harmless. Renovate ignores `prod/` entirely —
   this workflow is the sole owner of prod version bumps.

## Components

- [`renovate.json`](../renovate.json) — Renovate config (run by the Mend Renovate GitHub App).
- [`argocd-notifications-cm.yaml`](../src/core/argocd/manifests/argocd-notifications-cm.yaml) — the
  health gate: `on-deployed` trigger + GitHub webhook service.
- [`argocd-notifications-secret-externalsecret.yaml`](../src/core/argocd/manifests/argocd-notifications-secret-externalsecret.yaml)
  — GitHub token (from Infisical) for the webhook.
- [`promote-portfolio-prod.yml`](../.github/workflows/promote-portfolio-prod.yml) — the promotion PR.

## One-time setup

- Enable the **Mend Renovate GitHub App** on this repo and merge its onboarding PR.
- Add the Infisical secret `argocd-github-dispatch-token` (prod / `/cluster`): a GitHub PAT allowed
  to create repository_dispatch events on this repo (fine-grained: `Contents: Read and write`; or a
  classic PAT with the `repo` scope).

## Dev-overlay convention (required for the promotion flow)

The automation above is wired but **dormant until a `dev/` overlay exists** for the workload.
Create one mirroring `prod/`. For `portfolio`:

### `src/workloads/portfolio/dev/application.yaml`

Same multi-source shape as `prod/application.yaml`, but:

- `metadata.labels.env: dev`
- `spec.destination.namespace: portfolio-dev`
- value files: `base/values.yaml` + `dev/values.yaml`
- manifests path: `src/workloads/portfolio/dev/manifests`
- **subscribe to the health gate** via annotation:
  ```yaml
  metadata:
    annotations:
      notifications.argoproj.io/subscribe.on-deployed.github: ""
  ```

### `src/workloads/portfolio/dev/values.yaml`

Dev-specific overrides (hostname, replicas, etc.) plus the image tags. **Each tag must carry a
Renovate annotation comment** so the custom manager (which can't link the split repo/tag) tracks it:

```yaml
frontend:
  image:
    # renovate: datasource=docker depName=ghcr.io/michielvanherreweghe/portfolio-frontend
    tag: "2.2.0"
backend:
  image:
    # renovate: datasource=docker depName=ghcr.io/michielvanherreweghe/portfolio-backend
    tag: "1.0.0"
```

### Register it

Add `workloads/portfolio/dev/application.yaml` to [`src/kustomization.yaml`](../src/kustomization.yaml).
The `portfolio-dev` namespace is already allowed in
[`project.yaml`](../src/workloads/portfolio/project.yaml).
