---
name: deploy
description: Deploy project to production — commit, push, monitor GitHub Actions, verify server health, run pending fix scripts
argument-hint: "[--skip-fixes] [--dry-run]"
disable-model-invocation: true
---

## Dynamic context

- Branch: !`git branch --show-current`
- Deploy config: !`cat .flow/deploy.json 2>/dev/null || echo "NOT_FOUND"`
- Workflow: !`ls .github/workflows/deploy.yml 2>/dev/null || echo "NOT_FOUND"`
- Dockerfile.api: !`ls Dockerfile.api 2>/dev/null || echo "NOT_FOUND"`
- Dockerfile.web: !`ls Dockerfile.web 2>/dev/null || echo "NOT_FOUND"`

# Deploy to Production

Automated deployment workflow for projects with a `deploy.json` configuration file.

## Prerequisites

The deploy config is injected dynamically above. If `deploy.json` doesn't exist, Step -1 will invoke `/deploy-setup` to scaffold. After setup completes, resume from Step 0.

### deploy.json schema

Two schema versions are supported:

#### New schema (GHCR registry-based)

```json
{
  "ssh": {
    "host": "IP or hostname",
    "user": "SSH user",
    "project_dir": "/path/on/server"
  },
  "registry": {
    "provider": "ghcr",
    "org": "github-org",
    "images": ["api", "web"]
  },
  "deploy": {
    "runner_label": "server-infra",
    "compose_file": "docker-compose.prod.yml",
    "env_file": ".env.prod"
  },
  "health_check": {
    "url": "/api/health",
    "timeout": 120
  },
  "fixes": {
    "dir": "scripts/fixes",
    "tracking_file": "EXECUTED.md",
    "runner": "bun run",
    "apply_flag": "--fix"
  }
}
```

#### Legacy schema (build-on-server)

```json
{
  "ssh": {
    "host": "IP or hostname",
    "user": "SSH user",
    "project_dir": "/path/on/server"
  },
  "health_check": {
    "url": "https://example.com/health",
    "timeout": 120
  },
  "docker": {
    "compose_file": "docker-compose.prod.yml",
    "env_file": ".env.prod",
    "container": "api",
    "registry": "optional registry URL"
  },
  "fixes": {
    "dir": "scripts/fixes",
    "tracking_file": "EXECUTED.md",
    "runner": "bun run",
    "apply_flag": "--fix"
  }
}
```

**Schema detection**: If `registry` key is present → new schema (GHCR flow). If `docker` key is present → legacy schema (existing behavior). The skill adapts automatically.

#### Format 2: Multi-environment (branch-based)

```json
{
  "ssh": {
    "host": "IP or hostname",
    "user": "SSH user",
    "project_dir": "/path/on/server"
  },
  "environments": {
    "test": {
      "branch": "develop",
      "container": "api_test",
      "health_check": "https://api.test.example.com/",
      "runner_label": "server-infra"
    },
    "prod": {
      "branch": "main",
      "container": "api_prod",
      "health_check": "https://api.example.com/",
      "runner_label": "server-infra"
    }
  },
  "docker": {
    "compose_file": "docker-compose.yml"
  },
  "health_check": {
    "timeout": 120
  }
}
```

When `environments` is present, the skill automatically detects the current git branch and selects the matching environment.

## Workflow Steps

Execute these steps sequentially. At each step, use `AskUserQuestion` to confirm before proceeding to the next.

### Step -1: Quick setup check

Using the dynamic context injected above, verify deployment infrastructure exists:
- `deploy.json` is NOT "NOT_FOUND"
- Workflow is NOT "NOT_FOUND"
- At least one Dockerfile is NOT "NOT_FOUND"

If ANYTHING is NOT_FOUND: inform the user what is missing and invoke `/deploy-setup` to scaffold. After setup completes, resume from Step 0.

If everything exists: proceed to Step 0.

### Step 0: Detect environment

1. Read `deploy.json` from project root
1b. **Detect schema version**:
   - If `registry` key exists → new schema. Use `deploy.compose_file`, `deploy.env_file`, `deploy.runner_label`.
   - If `docker` key exists → legacy schema. Use `docker.compose_file`, `docker.env_file`, `docker.container`.
   - Display schema version detected.
2. Run `git branch --show-current` to get the current branch
3. **If `environments` key exists** (multi-environment mode):
   - Find the environment where `branch` matches the current git branch
   - If no match found, ask user which environment to deploy to
   - Store the selected environment's `container` and `health_check` for later steps
   - Display: "Deploying to **{env_name}** (branch: {branch}, container: {container})"
4. **If no `environments` key** (simple mode):
   - Use `docker.container` (or default to checking all containers)
   - Use `health_check.url` directly
   - Continue with legacy behavior
5. Adapt Step 1's branch comparison based on the target branch (e.g., `origin/develop..HEAD` for test, `origin/main..HEAD` for prod)

### Step 1: Check local state

1. Run `git status` to check for uncommitted changes
2. If there are changes:
   - Show the user a summary of modified/untracked files
   - Ask: "Do you want to commit these changes before deploying?"
   - If yes: use the `/commit` skill to create a proper commit
   - If no: warn that uncommitted changes won't be deployed and ask to confirm continuing
3. Run `git log origin/main..HEAD` to check for unpushed commits
4. If there are unpushed commits:
   - Show the commit list
   - Ask: "Push these commits to trigger deployment?"
   - If yes: run `git push`
   - If no: stop deployment

### Step 2: Monitor GitHub Actions

1. Check if a workflow exists: `gh workflow list`
2. If a deploy workflow exists:
   - Wait for the workflow run triggered by the push: `gh run list --workflow=deploy.yml --limit=1`
   - Monitor the run: `gh run watch <run-id>` (use Bash with timeout 600s)
   - If the run fails:
     - Show the logs: `gh run view <run-id> --log-failed`
     - Ask the user what to do (retry, investigate, abort)
   - If the run succeeds: continue to next step
3. If no workflow exists:
   - Inform the user there's no GitHub Actions workflow
   - Ask if they want to proceed with manual server verification anyway

**For new schema (GHCR)**: The workflow has TWO jobs:
1. "Build & Push Images" (cloud runner) — builds Docker images and pushes to GHCR
2. "Deploy to Server" (self-hosted runner) — pulls images and deploys

If job 1 fails → likely a code or Dockerfile issue. Show build logs.
If job 2 fails → likely a server or network issue. Suggest SSH investigation.

### Step 3: Verify server health

Using SSH info from `deploy.json` and environment detected in Step 0:

1. Ask: "Connect to {env_name} server to verify deployment?"
2. If yes:
   - SSH to the server and check the specific container status:
     ```bash
     # If container is known (from environment or docker.container):
     ssh {user}@{host} "docker ps --filter name={container} --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'"

     # Otherwise, check all containers via compose:
     ssh {user}@{host} "cd {project_dir} && docker compose -f {compose_file} ps"
     ```
   - Check the health endpoint (use `health_check` from the detected environment, or `health_check.url` in simple mode):
     ```bash
     curl -sf --max-time 10 {health_check_url}
     ```
   - If health check fails, check container logs for errors:
     ```bash
     ssh {user}@{host} "docker logs {container} --tail=30 --since=2m"
     ```
   - (Optional) Check Prisma migrations if applicable:
     ```bash
     ssh {user}@{host} "docker exec -T {container} npx prisma migrate status"
     ```
   - Report results clearly:
     - Environment: {env_name}
     - Container: {container} - running / stopped / restarting
     - Health check: OK / FAIL
     - Recent errors: none / list them

**For new schema**: After health check passes, verify the deployed image matches the expected tag:
```bash
ssh {user}@{host} "docker inspect {container} --format '{{.Config.Image}}'"
```
Confirm the image tag matches the commit that was pushed.

### Step 3b: Rollback (if health check failed)

If the health check in Step 3 fails after a successful workflow:

1. Show the last 50 lines of container logs
2. Ask: "The deployment seems unhealthy. Would you like to rollback to the previous version?"
3. If yes:
   - SSH to server
   - Find the previous image tag: `docker inspect {container} --format '{{.Config.Image}}'` (this shows the currently failing tag)
   - List available tags: `ssh {user}@{host} "docker images ghcr.io/{org}/{repo}-api --format '{{.Tag}}'"`
   - Ask the user which tag to rollback to
   - Update the image tag in compose and redeploy:
     ```bash
     ssh {user}@{host} "cd {project_dir} && sed -i 's|{current_image}|{rollback_image}|g' {compose_file} && docker compose -f {compose_file} --env-file {env_file} pull && docker compose -f {compose_file} --env-file {env_file} up -d"
     ```
   - Re-run health check
4. If no: report failure and stop

### Step 4: Run pending fix scripts (if configured)

Only if `deploy.json` has a `fixes` section AND the directory exists on the server:

1. SSH to server and list fix scripts:
   ```bash
   ssh {user}@{host} "cd {project_dir} && ls -la {fixes.dir}/"
   ```
2. Read the tracking file to find already-executed scripts:
   ```bash
   ssh {user}@{host} "cd {project_dir} && cat {fixes.dir}/{tracking_file} 2>/dev/null || echo 'No tracking file'"
   ```
3. Determine which scripts are pending (have dates newer than last executed, or not listed in tracking file)
4. If there are pending scripts:
   - List them with their filenames and dates
   - Ask: "Run these scripts in dry-run mode first?"
   - If yes: execute each in dry-run (no --fix flag):
     ```bash
     ssh {user}@{host} "cd {project_dir} && {runner} {fixes.dir}/{script_name}"
     ```
   - Show dry-run output for each script
   - Ask: "The dry-run results look correct. Apply these fixes for real?"
   - If yes: execute with apply flag:
     ```bash
     ssh {user}@{host} "cd {project_dir} && {runner} {fixes.dir}/{script_name} {apply_flag}"
     ```
   - Update the tracking file on the server after each successful execution
5. If no pending scripts: inform the user

### Step 5: Summary

Present a final deployment report:

```
Deployment Summary
------------------
Project:     {project name}
Environment: {env_name} ({branch})
Server:      {host}
Container:   {container}
Commit:      {commit hash} - {commit message}
Image:       {image tag if new schema}
Workflow:    {passed/failed/none}
Status:      {container status}
Health:      {OK / FAIL}
Fix scripts: {N applied / none pending / skipped}
```

## Arguments

- `--skip-fixes`: Skip step 4 (fix scripts)
- `--dry-run`: Go through all steps but don't actually push, execute scripts, or make changes. Show what would happen.

## Important notes

- ALWAYS use `AskUserQuestion` before any action that modifies state (push, run scripts, etc.)
- NEVER run fix scripts without showing dry-run first
- If SSH connection fails, suggest checking Tailscale/VPN connection
- If health check fails after successful workflow, check container logs for errors
- The health check URL in deploy.json is a path (e.g. `/api/health`). Determine the full URL from project context (domain in Caddyfile, .env.prod, or CLAUDE.md)
