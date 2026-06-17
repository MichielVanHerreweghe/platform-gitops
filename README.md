# platform-gitops

ArgoCD GitOps monorepo for the platform.

- `src/core/**` — core/platform workloads (cert-manager, cloudnative-pg, envoy-gateway, etc.).
- `src/workloads/**` — application workloads (e.g. `portfolio`).

## Keeping workloads up to date

Updates are automated — see [docs/automated-updates.md](docs/automated-updates.md). In short: core
charts get a PR for review; application workloads auto-update `dev` and, once dev is Healthy, open a
PR to promote the same version to `prod`.
