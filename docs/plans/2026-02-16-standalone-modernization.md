# Standalone Build Modernization Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Move standalone Docker image builds into the fundamento-cloud CI pipeline, simplify the standalone repo to pure deployment scaffolding, and auto-generate credentials on first boot.

**Architecture:** The fundamento-cloud workflow gains a matrix entry that builds a `RAILS_ENV=standalone` image and pushes it to `ghcr.io/ikigai-systems/fundamento`. The fundamento-standalone repo becomes a docker-compose file + env template + docs that reference pre-built images. A new Dockerfile target `standalone` and entrypoint logic handle credential auto-generation.

**Tech Stack:** GitHub Actions, Docker BuildKit, SOPS/age, Rails credentials, Docker Compose

---

### Task 1: Add `standalone` Dockerfile target in fundamento-cloud

**Files:**
- Modify: `/Users/pawel/Development/Ikigai-Systems/fundamento-cloud/Dockerfile:172-174`

**Step 1: Add the standalone target**

After the `e2e` stage and before the `production` stage, add a `standalone` target. This is an alias for `packaged` — the standalone conditional logic already lives in the `packaged` stage (lines 140-147) and activates when `RAILS_ENV=standalone`.

In `/Users/pawel/Development/Ikigai-Systems/fundamento-cloud/Dockerfile`, change:

```dockerfile
# Publish production as the default layer
FROM packaged AS production
```

to:

```dockerfile
# Standalone variant (credentials stripped, writable config/credentials, nano installed)
FROM packaged AS standalone

# Publish production as the default layer
FROM packaged AS production
```

**Step 2: Verify the build target works**

Run:
```bash
cd /Users/pawel/Development/Ikigai-Systems/fundamento-cloud
docker build --target standalone --build-arg RAILS_ENV=standalone --no-cache -f Dockerfile . 2>&1 | tail -5
```

Expected: Build completes (may fail at asset precompile without SOPS key, but the target should resolve).

Note: A full build requires SOPS secrets. Verifying the target resolves is sufficient for local testing. CI will do the full build.

**Step 3: Commit**

```bash
cd /Users/pawel/Development/Ikigai-Systems/fundamento-cloud
git add Dockerfile
git commit -m "Add standalone Dockerfile target for self-hosted builds"
```

---

### Task 2: Update docker-entrypoint for auto-credential generation

**Files:**
- Modify: `/Users/pawel/Development/Ikigai-Systems/fundamento-cloud/bin/docker-entrypoint`

**Step 1: Add credential auto-generation**

The current entrypoint:

```bash
#!/bin/bash -e

if [ "${1}" == "./bin/rails" ] && [ "${2}" == "server" ]; then
  ./bin/rails db:prepare
fi

exec "${@}"
```

Change to:

```bash
#!/bin/bash -e

# Auto-generate credentials for standalone on first boot
if [ "$RAILS_ENV" = "standalone" ] && [ ! -f config/credentials/standalone.yml.enc ]; then
  echo "First boot detected — generating standalone credentials..."
  EDITOR=true ./bin/rails credentials:edit -e standalone
fi

if [ "${1}" == "./bin/rails" ] && [ "${2}" == "server" ]; then
  ./bin/rails db:prepare
fi

exec "${@}"
```

Key points:
- `EDITOR=true` creates the credentials file without opening an editor (`true` is a no-op command)
- Only runs when `standalone.yml.enc` doesn't exist (first boot or fresh volume)
- Runs for ALL commands (server, good_job, rake tasks), not just `rails server`, because any Rails process needs credentials
- Runs before `db:prepare` because database setup may need credentials

**Step 2: Commit**

```bash
cd /Users/pawel/Development/Ikigai-Systems/fundamento-cloud
git add bin/docker-entrypoint
git commit -m "Auto-generate standalone credentials on first boot"
```

---

### Task 3: Extend CI build matrix in fundamento-cloud

**Files:**
- Modify: `/Users/pawel/Development/Ikigai-Systems/fundamento-cloud/.github/workflows/push-to-github-packages.yaml`

**Step 1: Add rails-env to matrix and standalone entry**

The current matrix (lines 26-33):

```yaml
    strategy:
      matrix:
        include:
          - image-name: fundamento-cloud
            context: .
            needs-secrets: true
          - image-name: blocknote-converter
            context: micro-services/blocknote-converter
            needs-secrets: false
```

Change to:

```yaml
    strategy:
      matrix:
        include:
          - image-name: fundamento-cloud
            context: .
            needs-secrets: true
            rails-env: production
            build-target: production
          - image-name: fundamento
            context: .
            needs-secrets: true
            rails-env: standalone
            build-target: standalone
          - image-name: blocknote-converter
            context: micro-services/blocknote-converter
            needs-secrets: false
            rails-env: ''
            build-target: ''
```

**Step 2: Update build-args to use matrix rails-env**

The current build step (lines 102-118) has:

```yaml
          build-args: |
            SOPS_VERSION=${{ env.SOPS_VERSION }}
            NODE_MAJOR=${{ env.NODE_MAJOR }}
```

Change to:

```yaml
          build-args: |
            SOPS_VERSION=${{ env.SOPS_VERSION }}
            NODE_MAJOR=${{ env.NODE_MAJOR }}
            ${{ matrix.rails-env && format('RAILS_ENV={0}', matrix.rails-env) || '' }}
          target: ${{ matrix.build-target || '' }}
```

Note: The `target` parameter is added. When empty string, `docker/build-push-action` builds the default target (last stage). For `fundamento-cloud` it targets `production`, for `fundamento` it targets `standalone`, for `blocknote-converter` it builds the default (which is `production` in that Dockerfile).

**Step 3: Commit**

```bash
cd /Users/pawel/Development/Ikigai-Systems/fundamento-cloud
git add .github/workflows/push-to-github-packages.yaml
git commit -m "Add standalone image to CI build matrix

Builds ghcr.io/ikigai-systems/fundamento alongside the existing
cloud and blocknote-converter images on every master merge."
```

---

### Task 4: Rewrite standalone docker-compose.yml

**Files:**
- Modify: `/Users/pawel/Development/Ikigai-Systems/fundamento-standalone/docker-compose.yml`

**Step 1: Rewrite docker-compose.yml**

Replace the entire contents with:

```yaml
x-rails-environment: &rails-environment
  DATABASE_URL: postgres://postgres:password@postgresql/postgres
  REDIS_URL: redis://redis
  HTTP_HOST: ${HTTP_HOST:-localhost:3000}
  RAILS_LOG_LEVEL: ${RAILS_LOG_LEVEL:-info}

services:
  postgresql:
    image: postgres:16
    environment:
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres:/var/lib/postgresql/data
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U postgres" ]
      interval: 5s
      timeout: 10s
      retries: 3
  redis:
    image: redis
    healthcheck:
      test: [ "CMD", "redis-cli", "ping" ]
      interval: 5s
      timeout: 10s
      retries: 3
  website:
    image: ghcr.io/ikigai-systems/fundamento:${FUNDAMENTO_VERSION:-latest}
    environment:
      <<: *rails-environment
    volumes:
      - credentials:/rails/config/credentials
      - storage:/rails/storage
    env_file:
      - env.standalone
    ports:
      - '${RAILS_PORT:-3000}:3000'
    depends_on:
      postgresql:
        condition: service_healthy
      redis:
        condition: service_healthy
  jobs:
    image: ghcr.io/ikigai-systems/fundamento:${FUNDAMENTO_VERSION:-latest}
    command: bundle exec good_job start
    environment:
      <<: *rails-environment
    volumes:
      - credentials:/rails/config/credentials
      - storage:/rails/storage
    env_file:
      - env.standalone
    depends_on:
      postgresql:
        condition: service_healthy
      redis:
        condition: service_healthy

volumes:
  postgres:
  credentials:
  storage:
```

Key changes from current:
- Removed `formula` service (formula-eval was removed from cloud)
- Removed `FORMULA_EVAL_MICROSERVICE_URL` environment variable
- Removed `EDITOR=nano` (no longer needed — credentials auto-generated)
- Added YAML anchor for shared environment config
- Added `storage` volume for ActiveStorage local files
- Added `HTTP_HOST` and `RAILS_LOG_LEVEL` environment variables
- Configurable port via `RAILS_PORT` env var
- Configurable image version via `FUNDAMENTO_VERSION` env var
- No exposed postgres/redis ports (internal networking only)

**Step 2: Validate compose syntax**

Run:
```bash
cd /Users/pawel/Development/Ikigai-Systems/fundamento-standalone
docker compose config --quiet
```

Expected: No output (valid syntax).

**Step 3: Commit**

```bash
cd /Users/pawel/Development/Ikigai-Systems/fundamento-standalone
git add docker-compose.yml
git commit -m "Rewrite docker-compose.yml for modernized standalone setup

Remove formula service, add YAML anchors, configurable ports and
version, storage volume, and shared environment config."
```

---

### Task 5: Clean up dead files in standalone repo

**Files:**
- Delete: `/Users/pawel/Development/Ikigai-Systems/fundamento-standalone/.github/workflows/push-to-github-packages.yaml`
- Delete: `/Users/pawel/Development/Ikigai-Systems/fundamento-standalone/dockerfiles/minio-init.sh`

**Step 1: Delete old workflow and minio init script**

```bash
cd /Users/pawel/Development/Ikigai-Systems/fundamento-standalone
git rm .github/workflows/push-to-github-packages.yaml
git rm dockerfiles/minio-init.sh
rmdir dockerfiles 2>/dev/null || true
rmdir .github/workflows 2>/dev/null || true
rmdir .github 2>/dev/null || true
```

**Step 2: Commit**

```bash
cd /Users/pawel/Development/Ikigai-Systems/fundamento-standalone
git commit -m "Remove build workflow and dead minio init script

Builds now happen in fundamento-cloud. The formula-eval and minio
services are no longer part of the standalone distribution."
```

---

### Task 6: Add .env.example to standalone repo

**Files:**
- Create: `/Users/pawel/Development/Ikigai-Systems/fundamento-standalone/.env.example`
- Modify: `/Users/pawel/Development/Ikigai-Systems/fundamento-standalone/.gitignore`

**Step 1: Create .env.example**

```env
# Fundamento Standalone Configuration
#
# Copy this file to .env and adjust values as needed:
#   cp .env.example .env
#
# These are optional — defaults work out of the box.

# Web server port (default: 3000)
# RAILS_PORT=3000

# Public hostname used for links in emails and redirects (default: localhost:3000)
# HTTP_HOST=localhost:3000

# Log level: debug, info, warn, error, fatal (default: info)
# RAILS_LOG_LEVEL=info

# Pin a specific Fundamento version instead of :latest
# See releases at: https://github.com/Ikigai-Systems/fundamento-standalone/releases
# FUNDAMENTO_VERSION=latest
```

**Step 2: Add .env to .gitignore**

```
.env
```

**Step 3: Commit**

```bash
cd /Users/pawel/Development/Ikigai-Systems/fundamento-standalone
git add .env.example .gitignore
git commit -m "Add .env.example for optional configuration"
```

---

### Task 7: Rewrite README.md

**Files:**
- Modify: `/Users/pawel/Development/Ikigai-Systems/fundamento-standalone/README.md`

**Step 1: Rewrite README.md**

```markdown
# Fundamento - Strong foundation for your internal collaboration

<p align="center">
  <a href="https://fundamento.it" target="_blank" align="center">
    <img src="https://res.cloudinary.com/fundamento/image/upload/v1734469016/fundamento_banner.webp" width="900" alt="Fundamento Banner">
  </a>
  <br>
</p>

---

## Quick Start

1. Clone this repository:
   ```
   git clone https://github.com/Ikigai-Systems/fundamento-standalone.git
   cd fundamento-standalone
   ```

2. (Optional) Edit `env.standalone` to customize the initial admin account.
   Defaults are `john@fundamento.it` / `secret!`.

3. Start Fundamento:
   ```
   docker compose up
   ```

4. Open `http://localhost:3000` and log in.

That's it — credentials are generated automatically on first boot.

---

## Architecture

Fundamento runs as a set of Docker containers:

| Service | Purpose |
|---------|---------|
| **website** | The main web application (Rails) |
| **jobs** | Background job worker (GoodJob) |
| **postgresql** | Database (PostgreSQL 16) |
| **redis** | Caching and real-time features |

---

## Configuration

### Admin account (`env.standalone`)

Edit before first start to set your admin credentials:

| Variable | Default |
|----------|---------|
| `FUNDAMENTO_ORGANIZATION` | Fundamento |
| `FUNDAMENTO_ADMIN_EMAIL` | john@fundamento.it |
| `FUNDAMENTO_ADMIN_FIRST_NAME` | John |
| `FUNDAMENTO_ADMIN_LAST_NAME` | Doe |
| `FUNDAMENTO_ADMIN_PASSWORD` | secret! |

### Environment variables (`.env`)

Copy `.env.example` to `.env` to customize:

| Variable | Default | Description |
|----------|---------|-------------|
| `RAILS_PORT` | 3000 | Web server port |
| `HTTP_HOST` | localhost:3000 | Public hostname for links/emails |
| `RAILS_LOG_LEVEL` | info | Log verbosity (debug/info/warn/error) |
| `FUNDAMENTO_VERSION` | latest | Pin a specific image version |

---

## Updating

Pull the latest images and restart:

```
docker compose pull
docker compose up -d
```

Database migrations run automatically on startup.

To pin a specific version, set `FUNDAMENTO_VERSION` in your `.env` file.

---

## Customizing Credentials

Credentials are auto-generated on first boot. To view or edit them later:

```
docker compose run --rm website bin/rails credentials:edit -e standalone
```

This opens the Nano editor. Press `Ctrl-X` to save and exit.

---

## Troubleshooting

### Port 3000 already in use

Set a different port in `.env`:
```
RAILS_PORT=3001
```

### Viewing logs

```
docker compose logs -f           # all services
docker compose logs -f website   # web application only
```

### Resetting everything

To start fresh (this destroys all data):
```
docker compose down -v
docker compose up
```

### Re-seeding admin account

After changing `env.standalone`, re-seed without losing other data:
```
docker compose exec website bin/rails db:seed:replant
```
**Warning:** This resets all data in the database.

---

## Documentation

https://docs.fundamento.it

---

<p align="center">
<a href="https://fundamento.it">Fundamento</a> &bull;
<a href="https://docs.fundamento.it">Docs</a> &bull;
<a href="https://fundamento.it/pricing">Pricing</a> &bull;
<a href="https://fundamento.it/terms">Terms</a> &bull;
<a href="https://fundamento.it/privacy">Privacy</a>
</p>
```

**Step 2: Commit**

```bash
cd /Users/pawel/Development/Ikigai-Systems/fundamento-standalone
git add README.md
git commit -m "Rewrite README with simplified install flow

Installation is now 3 steps: clone, (optional) edit env, docker compose up.
Adds architecture overview, configuration reference, and troubleshooting."
```

---

### Task 8: Update design doc to reflect final decisions

**Files:**
- Modify: `/Users/pawel/Development/Ikigai-Systems/fundamento-standalone/docs/plans/2026-02-16-standalone-modernization-design.md`

**Step 1: Update the design doc**

Update the design document to reflect:
- No blocknote-converter in standalone (removed per discussion — it runs in-process via the main image)
- 4 services not 5: postgresql, redis, website, jobs

**Step 2: Commit**

```bash
cd /Users/pawel/Development/Ikigai-Systems/fundamento-standalone
git add docs/plans/2026-02-16-standalone-modernization-design.md
git commit -m "Update design doc to reflect final decisions"
```

---

## Execution Order

Tasks 1-3 modify `fundamento-cloud` (can be done in one session).
Tasks 4-8 modify `fundamento-standalone` (can be done in one session).
Both groups are independent and can run in parallel.

## Post-Implementation

After all tasks are done:
1. Push fundamento-cloud changes to a branch, create PR
2. Push fundamento-standalone changes to a branch, create PR
3. Once cloud PR merges, the next master build will produce `ghcr.io/ikigai-systems/fundamento:master`
4. Test standalone locally: `docker compose up` in fundamento-standalone
5. Verify auto-credential generation works on first boot
