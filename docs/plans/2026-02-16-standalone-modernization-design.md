# Standalone Build Modernization Design

## Problem

The fundamento-standalone build system is broken and outdated:

- **Build failure**: The GitHub Actions workflow can't check out `fundamento-cloud` (PAT auth failure)
- **Dead services**: `formula-eval` was removed from cloud but standalone still references it
- **Manual process**: Releases require creating matching tags in both repos
- **Missing SOPS**: The standalone build doesn't pass the SOPS age key needed for asset precompilation
- **User friction**: Installation requires a confusing `rails credentials:edit` manual step
- **Drift**: Cloud docker-compose has improvements (YAML anchors, configurable ports, blocknote-converter) not reflected in standalone

## Decision: Build in fundamento-cloud

Following the Sentry/Supabase pattern, we move the standalone image build into the `fundamento-cloud` CI pipeline. The `fundamento-standalone` repo becomes pure deployment scaffolding (compose file, env templates, docs).

This eliminates cross-repo authentication, keeps builds automatically in sync, and shares the existing BuildKit cache.

## Design

### 1. CI/CD Pipeline (fundamento-cloud)

**Extend the build matrix** in `push-to-github-packages.yaml`:

| Entry | Image Name | Context | RAILS_ENV | Needs Secrets |
|-------|-----------|---------|-----------|---------------|
| existing | `fundamento-cloud` | `.` | production | yes |
| **new** | **`fundamento`** | **`.`** | **standalone** | **yes** |
| existing | `blocknote-converter` | `micro-services/blocknote-converter` | n/a | no |

The matrix gains a `rails-env` variable passed as a build arg. The standalone entry pushes to `ghcr.io/ikigai-systems/fundamento`.

**Add `standalone` Dockerfile target** (alias of `packaged`):

```dockerfile
FROM packaged AS standalone
```

This keeps the matrix clean: each entry's target name matches its image purpose.

**Auto-generate credentials on first boot** in `bin/docker-entrypoint`:

```bash
if [ "$RAILS_ENV" = "standalone" ] && [ ! -f config/credentials/standalone.yml.enc ]; then
  EDITOR=true bin/rails credentials:edit -e standalone
fi
```

Runs before `db:prepare`, only when no credentials file exists. Users can still customize credentials later.

### 2. Standalone Repository

**docker-compose.yml** — Rewritten to match cloud structure (minus minio):

Services:
- `postgresql` (postgres:16, internal only)
- `redis` (internal only)
- `website` (ghcr.io/ikigai-systems/fundamento:latest)
- `jobs` (same image, runs good_job)

Note: blocknote-converter is not needed as a separate service — it runs in-process via `BlocknoteConverterService` using Node.js built into the main image.

Key patterns from cloud:
- YAML anchors for shared environment config
- Configurable ports via env vars (RAILS_PORT)
- Health checks on infrastructure services
- Credentials volume for persistence

**Remove:**
- `.github/workflows/push-to-github-packages.yaml` (builds moved to cloud)
- `dockerfiles/minio-init.sh` (dead file)

**Keep:**
- `env.standalone` (admin seed configuration)

**Add:**
- `.env.example` (documents configurable env vars)

### 3. Documentation

**Simplified installation (3 steps):**

1. Clone the repo
2. (Optional) Edit `env.standalone` for admin credentials
3. `docker compose up`

**README.md sections:**
- Quick Start
- Architecture overview (4 services, what each does)
- Configuration reference (env vars table)
- Updating (`docker compose pull && docker compose up -d`)
- Customizing credentials
- Troubleshooting (port conflicts, logs, resetting data)

### 4. Image Naming

| Old | New |
|-----|-----|
| `ghcr.io/ikigai-systems/fundamento-standalone-website` | `ghcr.io/ikigai-systems/fundamento` |
| `ghcr.io/ikigai-systems/fundamento-standalone-formula` | removed (formula-eval is gone) |

## Changes by Repository

### fundamento-cloud (3 files)

1. `.github/workflows/push-to-github-packages.yaml` — add standalone matrix entry with `rails-env` variable
2. `Dockerfile` — add `FROM packaged AS standalone` target
3. `bin/docker-entrypoint` — add auto-credential generation for standalone

### fundamento-standalone (5 files)

1. `docker-compose.yml` — rewrite to match cloud structure, new image names
2. `README.md` — rewrite with simplified install flow and new sections
3. `.env.example` — new, documents configurable env vars
4. Delete `.github/workflows/push-to-github-packages.yaml`
5. Delete `dockerfiles/minio-init.sh`
