# platform-gitops

ArgoCD GitOps monorepo for the platform.

- `src/core/**` — core/platform workloads (cert-manager, cloudnative-pg, envoy-gateway, etc.).
- `src/workloads/**` — application workloads (e.g. `portfolio`).

## Keeping workloads up to date

Updates are automated — see [docs/automated-updates.md](docs/automated-updates.md). In short: core
charts get a Renovate PR for review. Application workloads stay manually versioned today; the
dev→prod auto-promotion flow is wired but dormant and activates per workload once that workload adds
a `dev/` overlay.
