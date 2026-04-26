---
name: deploy-setup
description: Use when a project needs deployment scaffolding — missing deploy.json, Dockerfiles, workflow, or docker-compose.prod.yml. Also invoked automatically by /deploy when setup is incomplete.
argument-hint: "[--check-only]"
disable-model-invocation: true
---

# Deploy Setup — Scaffold deployment infrastructure

Verify and scaffold the GHCR-based deployment pipeline for vibe-stack projects.

## Dynamic context

- deploy.json: !`cat .flow/deploy.json 2>/dev/null || echo "NOT_FOUND"`
- Workflow: !`cat .github/workflows/deploy.yml 2>/dev/null || echo "NOT_FOUND"`
- Dockerfile.api: !`head -5 Dockerfile.api 2>/dev/null || echo "NOT_FOUND"`
- Dockerfile.web: !`head -5 Dockerfile.web 2>/dev/null || echo "NOT_FOUND"`
- Compose images: !`grep "image:" docker-compose.prod.yml 2>/dev/null || echo "NOT_FOUND"`
- .dockerignore: !`head -1 .dockerignore 2>/dev/null || echo "NOT_FOUND"`
- GitHub user: !`gh api user --jq .login 2>/dev/null || echo "UNKNOWN"`
- Repo name: !`gh repo view --json nameWithOwner --jq .nameWithOwner 2>/dev/null || echo "UNKNOWN"`
- GitHub org: !`gh repo view --json owner --jq '.owner.login' 2>/dev/null || echo "UNKNOWN"`
- Org type: !`gh repo view --json owner --jq '.owner.type' 2>/dev/null || echo "UNKNOWN"`

## Checklist

Run through each check. If `$ARGUMENTS` contains `--check-only`, report status without creating files.

### 0. Server mode — integrated into step 1

There is no separate step 0. Server mode detection happens inside step 1 as part of the deploy.json questions. See the "Infrastructure" question below.

### 1. deploy.json

If NOT_FOUND or uses old schema (has `docker` key instead of `deploy`):

1. Ask the user for:
   - SSH host (IP or hostname)
   - SSH user (default: debian)
   - Project directory on server
   - **Infrastructure** : "PostgreSQL et Redis sont deja installes sur le serveur, ou il faut tout inclure dans le compose ?"
     - If already available → `shared` mode (external network `server-infra`, no DB/Redis in compose)
     - If not → `standalone` mode (internal network, DB + Redis + Caddy in compose)
   - **GitHub organization** : If no remote is configured or if the user wants to change it, ask whether the repo belongs to a personal account or an org. Use the org/user as `registry.org` in deploy.json. If the repo doesn't exist yet, create it with `gh repo create {org}/{repo} --private`.
   - Runner label for self-hosted deploy
   - Domain(s) for production (e.g. `pro.myapp.com` or `pro.myapp.com, app.myapp.com`)
   - Container prefix (default: repo name, used for container names and Caddy routing)
   - Health check path (default: /api/health)

   **For standalone mode only**, also ask:
   - PostgreSQL password (or generate a random 32-char alphanumeric one)
   - Redis password (or generate a random 32-char alphanumeric one)

If deploy.json already EXISTS and has no `caddy.mode`: check if it uses network `server-infra` (external) in docker-compose.prod.yml → shared. Otherwise → standalone. Ask the user to confirm.

2. Create `.flow/deploy.json` using this schema (create `.flow/` directory if needed):

   **Shared mode:**
   ```json
   {
     "ssh": { "host": "", "user": "debian", "project_dir": "" },
     "registry": { "provider": "ghcr", "org": "{github_user}", "images": ["api", "web"] },
     "deploy": { "runner_label": "", "compose_file": "docker-compose.prod.yml", "env_file": ".env.prod" },
     "caddy": { "mode": "shared", "domains": ["pro.myapp.com"], "container_prefix": "{repo_name}" },
     "health_check": { "url": "/api/health", "timeout": 120 }
   }
   ```

   **Standalone mode:**
   ```json
   {
     "ssh": { "host": "", "user": "debian", "project_dir": "" },
     "registry": { "provider": "ghcr", "org": "{github_user}", "images": ["api", "web"] },
     "deploy": { "runner_label": "", "compose_file": "docker-compose.prod.yml", "env_file": ".env.prod" },
     "caddy": { "mode": "standalone", "domains": ["pro.myapp.com"], "container_prefix": "{repo_name}" },
     "postgres": { "password": "{generated_or_provided}" },
     "redis": { "password": "{generated_or_provided}" },
     "health_check": { "url": "/api/health", "timeout": 120 }
   }
   ```

3. If old schema exists (has `docker` key), migrate: rename `docker` to `deploy`, add `registry`, `caddy`, and `runner_label`. If it was at root (`deploy.json`), move it to `.flow/deploy.json`.

### 2. Dockerfile.api

If NOT_FOUND or does NOT contain `FROM oven/bun` (old single-stage):

Read the template at `~/.claude/skills/deploy-setup/templates/Dockerfile.api.template` and create `Dockerfile.api`.

**CRITICAL — Docker build pattern (proven on all vibe-stack projects):**
- Stage 2 (build) MUST copy **only** `packages/shared/`, `apps/api/`, and `prisma/` — **NOT** `COPY . .` (copying `apps/web/` causes SWC to hang resolving unused deps)
- Shared package: compile with `bunx tsc -p tsconfig.json --module commonjs --moduleResolution node --outDir dist`, then copy the compiled output into `node_modules/{SHARED_PKG_NAME}/` (no symlinks)
- API build: generate a dedicated `nest-cli.docker.json` inline (avoids path issues with the repo's nest-cli.json) and run `bunx nest build --config nest-cli.docker.json`
- Always verify build output with `RUN ls apps/api/dist/main.js`
- `--mount=type=cache` is fine in templates (CI has buildx) — for local testing, see step 7.5
- **DO NOT** modify the repo's `nest-cli.json` or `tsconfig.json` for Docker — the Dockerfile is self-contained

Replace `{SHARED_PKG_NAME}` with the actual package name from `packages/shared/package.json` (e.g. `@toctoc/shared`).

Adapt if the project has different workspace structure (check `package.json` workspaces).

### 3. Dockerfile.web

If NOT_FOUND or does NOT contain `FROM oven/bun` (old single-stage):

Read the template at `~/.claude/skills/deploy-setup/templates/Dockerfile.web.template` and create `Dockerfile.web`.

### 4. .dockerignore

If NOT_FOUND:

Read the template at `~/.claude/skills/deploy-setup/templates/dockerignore.template` and create `.dockerignore`.

### 5. .env.build

If NOT_FOUND:

Create with defaults:
```
VITE_API_URL=/api
VITE_TRPC_URL=/trpc
```

Ask the user if these values are correct for their project.

### 6. docker-compose.prod.yml

If NOT_FOUND:
- If `caddy.mode` is `"shared"` (or not set): Read template at `~/.claude/skills/deploy-setup/templates/docker-compose.prod.template.yml`
- If `caddy.mode` is `"standalone"`: Read template at `~/.claude/skills/deploy-setup/templates/docker-compose.prod.standalone.template.yml`
- Create `docker-compose.prod.yml`, filling in values from deploy.json.

**IMPORTANT — PostgreSQL image selection (standalone mode):**
Scan Prisma migrations for required extensions:
```bash
grep -r "CREATE EXTENSION" prisma/migrations/ 2>/dev/null
```
Choose the image based on what's found:
- `postgis` → `postgis/postgis:16-3.5` (includes PostGIS)
- `vector` / `pgvector` → `pgvector/pgvector:pg16` (includes pgvector)
- Both `postgis` AND `vector` → `postgis/postgis:16-3.5` + add pgvector install in compose command
- Neither → `postgres:16-alpine` (lighter, no extensions needed)

The template uses `pgvector/pgvector:pg16` as default — **always override** based on actual extensions found.

For **standalone mode**, the template includes PostgreSQL, Redis, and Caddy services with the passwords from deploy.json. Ensure `DATABASE_URL` and `REDIS_URL` in the compose environment point to the compose service names (e.g. `postgres://user:pass@postgres:5432/dbname`, `redis://:pass@redis:6379`).

If EXISTS but uses `build:` instead of `image:`: Propose migration — replace `build:` with `image: ghcr.io/{org}/{repo}-{service}:latest`.

### 7. .github/workflows/deploy.yml

If NOT_FOUND or needs update:

- **Always use the standalone template** at `~/.claude/skills/deploy-setup/templates/deploy-workflow.standalone.template.yml` (self-contained inline workflow). Reusable workflows from private repos cause `startup_failure` on personal accounts and free orgs.
- The shared (reusable) template at `~/.claude/skills/deploy-setup/templates/deploy-workflow.template.yml` should only be used if the user explicitly requests it AND has a GitHub Team/Enterprise plan.

**IMPORTANT**: The `{PROJECT_DIR}` placeholder MUST be an absolute path (e.g. `/home/debian/zeendoc`), NOT use `~` — GitHub Actions does not expand `~` in `working-directory`.

Create `.github/workflows/deploy.yml`, filling in values from deploy.json.

### 7.5. Local Docker build validation

**MANDATORY** — Test Docker builds locally BEFORE any commit or push. This catches build errors instantly instead of wasting time on CI round-trips.

First check if buildx is available (needed for `--mount=type=cache` in Dockerfiles):
```bash
docker buildx version 2>/dev/null && BUILD_CMD="docker buildx build" || BUILD_CMD="docker build"
```

If buildx is NOT available, create a temp copy of the Dockerfile without `--mount=type=cache` lines for local testing (keep them in the committed Dockerfile — CI has buildx).

```bash
# Build API image
$BUILD_CMD -f Dockerfile.api -t {project}-api-test .

# Build Web image
$BUILD_CMD -f Dockerfile.web -t {project}-web-test .
```

**Then test that the API actually STARTS** (not just builds):
```bash
docker run --rm \
  -e DATABASE_URL="postgresql://postgres:test@localhost:5432/test" \
  -e REDIS_URL="redis://localhost:6379" \
  -e JWT_SECRET="$(openssl rand -hex 32)" \
  -e JWT_EXPIRES_IN="15m" \
  -e JWT_REFRESH_SECRET="$(openssl rand -hex 32)" \
  -e JWT_REFRESH_EXPIRES_IN="7d" \
  -e NODE_ENV=production -e PORT=3000 \
  {project}-api-test 2>&1 | head -30
```
Check the output for:
- `dependencies initialized` messages → OK, DI works
- `Cannot find module` → fix imports (deep imports of shared package, see step 7.6)
- `can't resolve dependencies` → fix missing `@Inject()` decorators or `import type` used for runtime DI tokens
- Env validation errors (Zod) → OK, means the app starts, just needs correct env vars

**A build that succeeds but crashes at runtime is NOT validated.** Both build AND startup must pass.

**If a build fails:**
1. Read the error output carefully
2. Fix the Dockerfile or source code
3. Rebuild locally until it passes
4. Only then proceed to the next steps

**If Docker is not available locally** (rare on macOS with Colima/Docker Desktop):
- Warn the user: "Docker n'est pas disponible localement. Les builds seront valides uniquement en CI — risque d'echecs."
- Use `AskUserQuestion` to confirm proceeding without local validation.

**Do NOT skip this step.** Every failed CI build wastes 5-10 minutes. A local build takes the same time but gives instant feedback.

### 7.6. Check for deep imports of shared package

**IMPORTANT** — Deep imports like `@xxx/shared/src/schemas/foo` work in dev (Bun/tsx resolves TS sources) but **break in Docker** where shared is compiled to `dist/`.

Search for deep imports:
```bash
grep -r "@{SHARED_PKG_NAME}/src/" apps/api/src/ --include="*.ts"
```

If found, replace them with barrel imports from `@{SHARED_PKG_NAME}`. Verify the barrel (`packages/shared/src/index.ts`) exports everything needed — add missing exports if necessary.

### 8. packages/shared build script

If `packages/shared/package.json` exists and does NOT have a `build` script:

Add `build` and `clean` scripts for **local dev** usage:

```json
"build": "tsc --outDir dist --declaration",
"clean": "rm -rf dist"
```

**DO NOT modify `packages/shared/tsconfig.json`** — the Dockerfile compiles shared with its own flags (`--module commonjs --moduleResolution node`), so the repo tsconfig can stay as ESNext/bundler for local dev with Vite/Bun.

### 8.5. NestJS nest-cli.json — DO NOT MODIFY

**DO NOT fix paths** in `apps/api/nest-cli.json`. The Dockerfile generates its own `nest-cli.docker.json` with correct paths (see step 2). The repo's nest-cli.json is only used for local dev (`bun run dev`) where paths may differ.

### 9. GitHub Actions Runner

This step is maximally automated. Follow sub-steps in order, skipping ahead when possible.

#### 9a. Check if runner exists for this repo or org

First check repo-level runners:
```bash
gh api repos/{owner}/{repo}/actions/runners --jq '.runners[] | .name + " (" + .status + ") [" + ([.labels[].name] | join(", ")) + "]"'
```

If no repo-level runner found, also check **org-level runners** (common with shared runners on an org):
```bash
gh api orgs/{owner}/actions/runners --jq '.runners[] | .name + " (" + .status + ") [" + ([.labels[].name] | join(", ")) + "]"' 2>/dev/null
```
Note: org-level runner API may fail with 403 if the user lacks `admin:org` permission — that's OK, fall back to checking the server directly (step 9b).

Also check the server directly for a runner registered to the org:
```bash
ssh {ssh_user}@{ssh_host} "test -f ~/actions-runner/.runner && python3 -c \"import json; f=open('/home/debian/actions-runner/.runner','rb'); data=f.read().lstrip(b'\\xef\\xbb\\xbf'); d=json.loads(data); print(d.get('agentName','?'), '->', d.get('gitHubUrl','?'))\" || echo 'NO_RUNNER'"
```
**Note:** The `.runner` file may have a UTF-8 BOM — always strip it before parsing JSON.

If a runner is found (at repo, org, or server level) with the matching label from deploy.json AND status is `"online"` → **DONE**, skip to step 10.

#### 9b. Check if runner exists on server (possibly registered to another repo)

```bash
ssh {ssh_user}@{ssh_host} "test -f ~/actions-runner/.runner && cat ~/actions-runner/.runner | python3 -c 'import sys,json; d=json.load(sys.stdin); print(f\"{d[\"agentName\"]} -> {d[\"gitHubUrl\"]}\")' || echo 'NO_RUNNER'"
```

If a runner exists but is registered to another repo:
- Tell the user: "Un runner existe deja sur ce serveur mais il est enregistre pour un autre repo. Tu peux le partager en allant dans GitHub Settings → Actions → Runners et en ajoutant ce repo. Sinon, on peut en installer un nouveau dans `~/actions-runner-{project}`."
- Use `AskUserQuestion` to let the user choose (share existing / install new).
- If sharing, update the `runner_dir` to `~/actions-runner`. If installing new, use `~/actions-runner-{project}`.

If `NO_RUNNER` → proceed to 9c.

#### 9c. Check prerequisites on server

```bash
ssh {ssh_user}@{ssh_host} "echo '=== Docker ===' && docker --version 2>/dev/null || echo 'NOT_INSTALLED' && echo '=== Docker Compose ===' && docker compose version 2>/dev/null || echo 'NOT_INSTALLED' && echo '=== Git ===' && git --version 2>/dev/null || echo 'NOT_INSTALLED'"
```

If Docker is missing, install it:
```bash
ssh {ssh_user}@{ssh_host} "curl -fsSL https://get.docker.com | sudo sh && sudo usermod -aG docker \$USER"
```

Warn the user: "L'utilisateur doit se reconnecter en SSH pour que le groupe `docker` prenne effet."

If Git is missing, install it:
```bash
ssh {ssh_user}@{ssh_host} "sudo apt-get update && sudo apt-get install -y git"
```

#### 9d. Generate runner registration token

Try the API first (works if the user has admin scope on the repo):
```bash
gh api repos/{owner}/{repo}/actions/runners/registration-token --jq .token
```

**If this succeeds** → use the token directly, proceed to 9e. No manual step needed.

**If this fails (403/404)** → fall back to the manual approach:

1. Generate the direct URL:
   `https://github.com/{owner}/{repo}/settings/actions/runners/new?arch=x64&os=linux`

2. Tell the user:
   > "Je ne peux pas generer le token automatiquement (permissions admin requises)."
   > "Ouvre ce lien et copie le token affiche dans la section 'Configure' :"
   > `https://github.com/{owner}/{repo}/settings/actions/runners/new?arch=x64&os=linux`
   > "Colle le token ici :"

3. Use `AskUserQuestion` to get the token from the user.

#### 9e. Install and configure runner

Once the token is available (auto or manual):

```bash
# Detect architecture
ARCH=$(ssh {ssh_user}@{ssh_host} "uname -m")
# Map: x86_64 → x64, aarch64 → arm64

# Get latest runner version
RUNNER_VERSION=$(gh api repos/actions/runner/releases/latest --jq .tag_name | sed 's/v//')

# Determine runner directory (default: ~/actions-runner, or ~/actions-runner-{project} if sharing)
RUNNER_DIR={runner_dir}

# Install and configure
ssh {ssh_user}@{ssh_host} "mkdir -p ${RUNNER_DIR} && cd ${RUNNER_DIR} \
  && curl -sL https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-${ARCH}-${RUNNER_VERSION}.tar.gz | tar xz \
  && ./config.sh --url https://github.com/{owner}/{repo} --token {TOKEN} --labels {runner_label} --name $(hostname)-runner --unattended --replace \
  && sudo ./svc.sh install \
  && sudo ./svc.sh start"
```

#### 9f. Verify runner is online

```bash
sleep 5
gh api repos/{owner}/{repo}/actions/runners --jq '.runners[] | .name + " (" + .status + ")"'
```

If the runner shows as `"online"` → success.

If not, troubleshoot:
```bash
ssh {ssh_user}@{ssh_host} "cd {runner_dir} && sudo ./svc.sh status"
```
Show the output and help the user debug.

### 9.5. Server preparation

Prepare the server for deployment. **This step applies to BOTH shared and standalone modes.**

#### 9.5a. Create project directory

```bash
ssh {ssh_user}@{ssh_host} "mkdir -p {project_dir}"
```

#### 9.5b. Check if `.env.prod` exists on server

```bash
ssh {ssh_user}@{ssh_host} "test -f {project_dir}/.env.prod && echo 'ENV_EXISTS' || echo 'ENV_MISSING'"
```

If `ENV_MISSING`, generate and deploy `.env.prod`:

**For shared mode** (DB/Redis on server-infra):
1. Read credentials from server-infra:
   ```bash
   ssh {ssh_user}@{ssh_host} "grep -E 'POSTGRES_PASSWORD|POSTGRES_USER' ~/server-infra/.env"
   ssh {ssh_user}@{ssh_host} "docker inspect server-redis --format '{{range .Config.Env}}{{println .}}{{end}}' | grep REDIS_PASSWORD"
   ```
2. Create the database if it doesn't exist:
   ```bash
   ssh {ssh_user}@{ssh_host} "docker exec server-postgres psql -U postgres -c 'CREATE DATABASE {project_name};'" 2>/dev/null || true
   ```
3. Generate JWT secrets:
   ```bash
   JWT_SECRET=$(openssl rand -hex 32)
   JWT_REFRESH=$(openssl rand -hex 32)
   ```
4. Build and deploy `.env.prod`:
   ```env
   # Database (server-infra PostgreSQL)
   DATABASE_URL=postgresql://postgres:{pg_password}@server-postgres:5432/{project_name}?schema=public

   # Redis (server-infra Redis)
   REDIS_URL=redis://:{redis_password}@server-redis:6379

   # JWT Authentication
   JWT_SECRET="{jwt_secret}"
   JWT_EXPIRES_IN="15m"
   JWT_REFRESH_SECRET="{jwt_refresh}"
   JWT_REFRESH_EXPIRES_IN="7d"

   # Application
   NODE_ENV=production
   PORT=3000
   ```
5. Also check `.env.example` for project-specific variables (MINIO, LITELLM, etc.) and add them with appropriate server-infra service names (e.g. `server-minio`, `server-litellm`).

**For standalone mode** (DB/Redis in compose):
```env
# Database (compose service)
DATABASE_URL=postgresql://postgres:{postgres_password}@postgres:5432/{project_name}

# Redis (compose service)
REDIS_URL=redis://:{redis_password}@redis:6379

# Application
NODE_ENV=production
PORT=3000
```

Push to server via heredoc (NOT scp — avoids temp files):
```bash
ssh {ssh_user}@{ssh_host} "cat > {project_dir}/.env.prod << 'EOF'
{content}
EOF"
```

#### 9.5c. Verify server readiness

```bash
ssh {ssh_user}@{ssh_host} "ls -la {project_dir}/.env.prod && echo 'ENV_OK' || echo 'ENV_MISSING'"
```

### 10. GHCR access

```bash
gh api 'user/packages?package_type=container' --jq '.[].name' 2>/dev/null
```

If this fails (missing `read:packages` scope):

**Always use `gh auth refresh` with device code flow** — it works whether `GH_TOKEN` is set or not:

```bash
GH_TOKEN="" gh auth refresh -h github.com -s read:packages,write:packages
```

This triggers the device code flow:
1. A one-time code is displayed (e.g. `9314-CBE8`)
2. Tell the user to open https://github.com/login/device and enter the code
3. Wait for authentication to complete

After success, verify with the `GH_TOKEN=""` prefix to use the updated keyring token:
```bash
GH_TOKEN="" gh api 'user/packages?package_type=container' --jq '.[].name' 2>/dev/null
```

**Note:** If `GH_TOKEN` is set in the environment, the updated scopes are stored in the keyring token. The workflow on GitHub Actions uses `GITHUB_TOKEN` (not `GH_TOKEN`), so this has no impact on CI. Locally, `gh` commands that need packages scope should be prefixed with `GH_TOKEN=""`.

### 11. Caddy reverse proxy

Behavior depends on `caddy.mode` in deploy.json:

#### Mode `shared` (default — server-infra Caddy)

Read the template at `~/.claude/skills/deploy-setup/templates/caddyfile-block.template`.

Generate the Caddy block by replacing `{PROJECT_NAME}`, `{DOMAINS}`, `{CONTAINER_PREFIX}` from deploy.json.

Check if the block already exists:
```bash
grep "{first_domain}" ~/Dev/server-infra/caddy/Caddyfile 2>/dev/null
```

- If **found**: Display "Caddy block already exists for {domain}" — no action needed.
- If **NOT found** and `~/Dev/server-infra/` exists locally — **execute all steps automatically, do NOT just list commands**:
  1. Append the generated block to `~/Dev/server-infra/caddy/Caddyfile` (before the template comment block)
  2. Commit and push server-infra:
     ```bash
     cd ~/Dev/server-infra && git add caddy/Caddyfile && git commit -m "feat(caddy): add {project_name} reverse proxy block" && git push origin main
     ```
  3. Pull on server and **restart** Caddy via SSH (read ssh config from deploy.json).
     **IMPORTANT**: Use `docker compose restart caddy` instead of `caddy reload` — `caddy reload` may report "config is unchanged" if the Caddyfile was updated outside the running process. A full restart forces Caddy to re-read the config and triggers certificate provisioning for new domains.
     ```bash
     ssh {ssh_user}@{ssh_host} "cd ~/server-infra && git pull && docker compose restart caddy"
     ```
  4. Wait 15 seconds for SSL certificate provisioning (new domains need ACME challenge), then verify:
     ```bash
     sleep 15 && curl -s --max-time 10 https://{first_domain}/api/health 2>&1
     ```
  5. If curl fails (SSL not ready yet), check Caddy logs for the domain:
     ```bash
     ssh {ssh_user}@{ssh_host} "docker logs server-caddy --since=30s 2>&1 | grep '{first_domain}'"
     ```
     If "authorization finalized" or "certificate obtained" appears, the cert is being provisioned — tell the user to wait a minute and retry. If no mention of the domain, Caddy may not have detected the config change — restart again.
- If **NOT found** and `~/Dev/server-infra/` does NOT exist locally: Display the generated block and tell the user to add it manually.

#### Mode `standalone` (Caddy in project compose)

Read the template at `~/.claude/skills/deploy-setup/templates/Caddyfile.standalone.template`.

Generate a `Caddyfile` at the project root by replacing `{DOMAINS}` and `{CONTAINER_PREFIX}` from deploy.json.

If the `Caddyfile` already exists, verify it contains the correct domain — update if needed.

The standalone compose template already includes the Caddy service (ports 80/443, volume-mounts the Caddyfile). No server-infra interaction needed.

## Summary

After all checks, display:

```
Deploy Setup — Status
---------------------
Server mode:           shared / standalone
GitHub remote:         OK ({org}/{repo}) / X not configured
deploy.json:           OK created / OK exists / X missing
Dockerfile.api:        OK created / OK exists (multi-stage) / X old format
Dockerfile.web:        OK created / OK exists (multi-stage) / X old format
.dockerignore:         OK created / OK exists / X missing
.env.build:            OK created / OK exists / X missing
docker-compose.prod:   OK created / OK using images / X using build
workflow:              OK created / OK exists (reusable) / X old format
Local build API:       OK passed / X failed (fix before push!)
Local build Web:       OK passed / X failed (fix before push!)
shared build:          OK has build script / X missing
Runner:                OK online ({label}) / OK installed / X not found
Server preparation:    OK ready (.env.prod + DB) / X needs setup
GHCR access:           OK / X needs setup
Caddy:                 OK exists ({domain}) / OK added / X needs manual setup
```
