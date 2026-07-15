# ohdat/actions

Org-wide reusable GitHub Actions for Ohdat services.

| Workflow | Purpose |
|----------|---------|
| [`docker-ci.yml`](.github/workflows/docker-ci.yml) | Multi-arch Docker build → **GHCR** |
| [`go-ci.yml`](.github/workflows/go-ci.yml) | GoReleaser + Docker build → **GHCR** |
| [`node-ci.yml`](.github/workflows/node-ci.yml) | Node build + Docker → **GHCR** |
| [`helm-cd.yml`](.github/workflows/helm-cd.yml) | Helm deploy to **Civo K3s** via **OCI charts** |
| [`helm-cron-cd.yml`](.github/workflows/helm-cron-cd.yml) | Same for CronJob charts |

## Defaults (2026)

| Item | Value |
|------|--------|
| Container registry | `ghcr.io/<owner>/<repo>` |
| Kubernetes | **Civo** cluster `ele-prod` / region `nyc1` |
| Helm charts | [`ohdat/helm-temp`](https://github.com/ohdat/helm-temp) → `oci://ghcr.io/ohdat/charts/*` |

> **Breaking (from older actions):** Docker Hub and AWS EKS are no longer used. Callers must pass `CIVO_TOKEN` for deploy workflows. Image outputs are now `ghcr.io/...` refs.

---

## docker-ci

```yaml
jobs:
  build:
    uses: ohdat/actions/.github/workflows/docker-ci.yml@master
    with:
      VERSION: v0.1.${{ github.run_number }}${{ github.ref != 'refs/heads/master' && '-dev' || '' }}
    secrets:
      GIT_TOKEN: ${{ secrets.GIT_TOKEN }}  # optional private deps
```

**Outputs:** `image` (e.g. `ghcr.io/ohdat/myapp:v0.1.12-dev`), `version`

---

## go-ci

```yaml
jobs:
  ci:
    uses: ohdat/actions/.github/workflows/go-ci.yml@master
    with:
      VERSION: v0.1.${{ github.run_number }}${{ github.ref != 'refs/heads/master' && '-dev' || '' }}
    secrets:
      GIT_TOKEN: ${{ secrets.GIT_TOKEN }}
```

Requires `go.mod` and a root `Dockerfile` (typically copies `dist/gomain_linux_*` from GoReleaser).

---

## node-ci

```yaml
jobs:
  build:
    uses: ohdat/actions/.github/workflows/node-ci.yml@master
    with:
      VERSION: v0.1.${{ github.run_number }}
      NODE_VERSION: "20"
      BUILD_COMMANDS: "npm ci && npm run build"
    secrets:
      GIT_TOKEN: ${{ secrets.GIT_TOKEN }}
```

---

## helm-cd (Civo + OCI)

```yaml
jobs:
  deploy:
    needs: ci
    uses: ohdat/actions/.github/workflows/helm-cd.yml@master
    with:
      environment: produce          # GitHub Environment name
      namespace: backend
      cluster: ele-prod             # optional, default
      region: nyc1                  # optional, default
      helm: http-myapp service      # "<release> <chart>"
      chart-version: "1.2.1"        # pin service chart
      set: >-
        image.repository=${{ needs.ci.outputs.image }},
        resources=null,
        autoscaling.enabled=false,
        gateway.enabled=true,
        gateway.create=false,
        gateway.existingGateway=istio-system/elevatrix-gateway,
        gateway.hosts[0]=myapp.elevatrix.xyz,
        service.port=8080
    secrets:
      CIVO_TOKEN: ${{ secrets.CIVO_TOKEN }}
      GIT_TOKEN: ${{ secrets.GIT_TOKEN }}   # optional, private OCI charts
```

### Chart names (`oci://ghcr.io/ohdat/charts/<name>`)

| Chart | Typical version | Use |
|-------|-----------------|-----|
| `service` | `1.2.1` | HTTP + Istio VirtualService |
| `grpc` | `1.1.0` | gRPC |
| `command` | `1.1.0` | long-running command |
| `cronjob` | `1.1.0` | CronJob |
| `socket` / `tcpservice` / `website` / … | `1.1.0` | see helm-temp |

Legacy chart refs like `ohdat/service` are accepted and mapped to OCI `service`.

### elevatrix.xyz gateway (recommended)

```yaml
gateway.enabled=true
gateway.create=false
gateway.existingGateway=istio-system/elevatrix-gateway
gateway.hosts[0]=<subdomain>.elevatrix.xyz
```

---

## helm-cron-cd

Same as `helm-cd`, plus `schedule` (default `0 0 * * *`) passed as helm `--set schedule=...`.

```yaml
jobs:
  deploy:
    uses: ohdat/actions/.github/workflows/helm-cron-cd.yml@master
    with:
      environment: produce
      namespace: backend
      helm: renew-myapp cronjob
      chart-version: "1.1.0"
      schedule: "0 0 * * *"
      set: "image.repository=ghcr.io/ohdat/myapp:v0.1.1,resources=null"
    secrets:
      CIVO_TOKEN: ${{ secrets.CIVO_TOKEN }}
```

---

## Required secrets (per consumer repo)

| Secret | Used by | Notes |
|--------|---------|--------|
| `CIVO_TOKEN` | helm-cd, helm-cron-cd | Civo API token with cluster access |
| `GIT_TOKEN` | go-ci, docker-ci, helm (optional) | PAT: private modules / `read:packages` for private charts |

GHCR push uses the job `GITHUB_TOKEN` (`packages: write`). Ensure org/repo package permissions allow it.

---

## Migration from old actions

| Old | New |
|-----|-----|
| Docker Hub (`DOCKERHUB_TOKEN`, `ohdat/...`) | GHCR (`ghcr.io/ohdat/...`, no Docker Hub secret) |
| AWS EKS (`AWS_*`, `kubename: ohdat-prod`) | Civo (`CIVO_TOKEN`, `cluster: ele-prod`) |
| `https://ohdat.github.io/helm-template/` | `oci://ghcr.io/ohdat/charts` ([helm-temp](https://github.com/ohdat/helm-temp)) |
| CloudFront `cacheid` | Removed (no-op if still passed) |

Old input/secret names that are still declared as **optional** so existing callers do not fail schema validation: `DOCKERHUB_*`, `AWS_*`, `kubename`, `cacheid`. They are ignored at runtime.
