# AGENTS.md

Manifest-only Flux CD GitOps repo — no code, no build/lint/test tooling.
Flux watches `main` on `github.com/mrunesson/gitops`; commit + push is the entire deploy flow (no CI, no PR gate).

## Layout

- `clusters/<cluster-name>/` — the exact tree Flux kustomize-applies to that cluster; one directory per cluster. Currently only `nuc` (vanilla Kubernetes, dev, `cluster.local`, sized `small`).
- `clusters/nuc/flux-system/flux-instance.yaml` — single source of truth for the Flux installation (Flux Operator CR `FluxInstance`; must stay named `flux` in `flux-system`).
- `clusters/nuc/flux-system/flux-runtime-info.yaml` — ConfigMap with per-cluster vars `CLUSTER_NAME=nuc`, `ENVIRONMENT=dev`, `CLUSTER_DOMAIN=cluster.local`.
- App workloads do NOT belong in `flux-system/` (control plane); each app gets its own directory under `clusters/<cluster>/`.

## How sync works (read before adding anything)

The FluxInstance `spec.sync` (this repo, path `clusters/nuc`) makes the operator create a `GitRepository` + root Kustomization, both named `flux-system`, which kustomize-applies the **whole** `clusters/nuc` tree (`prune: true`, 10m interval).

- New app: create `clusters/nuc/<namespace>/kustomization.yaml` + manifests **and** list the directory in `clusters/nuc/kustomization.yaml` `resources:` — kustomize ignores unlisted directories, so an unlisted app is never applied and never pruned.
- Validate before pushing: `kustomize build clusters/nuc` (optionally `kubectl apply -k clusters/nuc --dry-run=server`).

## Gotchas

- The in-cluster root `flux-system/flux-system` Kustomization and `flux-system` GitRepository are **owned by the Flux Operator** (`app.kubernetes.io/managed-by: flux-operator`, `kustomize.toolkit.fluxcd.io/ssa: Ignore`, `prune: Disabled`) — never commit or hand-edit them; drive changes via `flux-instance.yaml` (`spec.sync`, `spec.kustomize.patches`).
- Current sync inventory: flux-system (FluxInstance + `flux-runtime-info`), `cert-manager`, `cnpg-system`, `keycloak`.
- Tenant pattern (copy for new apps): `namespace.yaml` + `flux-rbac.yaml` (ServiceAccount `flux` — **mandatory**, multitenant controllers impersonate it) + chart source + `helmrelease.yaml`. Chart sources: cert-manager & keycloak use `OCIRepository`, cnpg-system uses `HelmRepository` (cloudnative-pg.io/charts). Every HelmRelease must set `serviceAccountName: flux`; `OCIRepository` takes that field too, but v1 `HelmRepository` has **no** `spec.serviceAccountName` — adding it fails the dry-run.
- RBAC per tenant differs — check, don't assume: cert-manager and cnpg-system still bind `flux` to **cluster-admin** (legacy, dev single-node cluster). keycloak was deliberately reduced (commit "keycloak reduce flux permissions") to just the ServiceAccount, no ClusterRoleBinding — verified working live because the chart deploys only namespaced resources. Prefer the minimal keycloak shape for new tenants.
- The letsencrypt ClusterIssuers (`letsencrypt-staging`/`letsencrypt-prod`) solve HTTP-01 via ingress class `traefik`; keycloak's ingress already references `letsencrypt-prod`, but no traefik workload exists in this repo — certificates will not issue until ingress/traefik lands.
- Multitenancy lockdown is on (`multitenant: true`, default tenant SA `flux`, NetworkPolicy on): Flux controllers impersonate the `flux` SA and cross-namespace references are blocked. Each tenant namespace needs its own ServiceAccount + RoleBinding + source credentials; to use `flux-runtime-info` vars in a namespace, copy the CM there (ResourceSet `copyFrom`) and reference the **local** copy in `postBuild.substituteFrom` — pointing at `flux-system` will not work.
- `flux-runtime-info` carries `reconcile.fluxcd.io/watch: Enabled` (changes re-reconcile dependents) and `kustomize.toolkit.fluxcd.io/ssa: "Merge"` — preserve both when editing.
- To add a cluster: the Flux Operator must be installed there first (Helm/Terraform, outside this repo), then copy `clusters/nuc/` → `clusters/<name>/` and adjust the sync `path` + `flux-runtime-info` vars.

## Local-only tooling

- `.gitignore` covers only `.agents/` and `.opencode/` (MCP servers: `flux-operator-mcp` with write access, `kubernetes`, `flux-schema-catalog`, `google-search`, `safari-mcp`) — never commit them.
- `.junie/`, `.output.txt`, `.output.json` are local artifacts that are **not** gitignored — do not `git add` them; add to `.gitignore` or clean up instead.
- Prefer `flux-schema-catalog` MCP for validating Flux/K8s manifests and `flux-operator-mcp` / `kubernetes` MCP for live cluster state over guessing.
