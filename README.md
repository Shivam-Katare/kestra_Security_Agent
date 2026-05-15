# Dev Security Agent

Modern CI/CD pipelines can still miss dangerous changes that only become visible after deployment, such as poisoned dependencies, tampered lockfiles, or suspicious outbound browser requests. Dev Security Agent solves that gap by running layered post-deploy security checks, sending one final Slack alert, and optionally rolling production back to the previous safe Vercel deployment.

This project is a multi-flow production security orchestration setup built with Kestra for modern web applications. It combines dependency scanning, browser runtime monitoring, AI-based result summarization, Slack alerting, and optional automated rollback.

**Optional rollback**: if the pipeline detects a critical security issue, it can automatically reassign the production alias to the previous safe Vercel deployment.

## The problem it solves

Traditional CI/CD checks are often not enough on their own.

A deployment may still reach production even when:

- a risky dependency is introduced through `package.json` or `package-lock.json`
- a lockfile is silently modified in a suspicious way
- malicious or unapproved runtime scripts start making outbound calls after deployment
- the issue appears only after the application is live

That means the dangerous version can already be serving users before a team notices it.

Dev Security Agent is designed to catch those cases by running layered checks after deploy or on a fixed schedule, then taking action quickly:

- refresh the latest package files
- scan dependencies
- inspect browser runtime network behavior
- send one final Slack alert
- optionally roll back production to the previous safe deployment

## What it does

The pipeline checks two major attack surfaces in a deployed frontend application:

- `package.json` and `package-lock.json` for dependency and supply chain risk
- live browser runtime behavior for unexpected outbound requests or suspicious domains

After both checks complete, the main orchestration flow combines the results, sends them to Gemini for a clean security summary, posts one final alert to Slack, and can optionally trigger a rollback on Vercel.

## Why orchestration is needed

Attackers increasingly target CI/CD and frontend delivery pipelines through:

- malicious or poisoned dependencies
- tampered lockfiles
- runtime script injection after deploy
- hidden outbound requests to attacker-controlled domains

A single scan is not enough. This project uses orchestration so both build-time and runtime risks are checked together before a deployment is treated as safe.

## How it works

This diagram shows how Kestra orchestrates triggers, scans, summary generation, Slack alerting, and rollback.

## Project structure

```text
KESTRA_SECURITY_AGENT/
├── kestra/
│   ├── security_pipeline_main_prod.yml
│   ├── security_scan_layer1_prod.yml
│   ├── security_scan_layer2_prod.yml
│   └── upload_lockfile_prod.yml
├── docker-compose.yaml
├── .env.example
└── README.md
```

## Flows

### 1. `upload_lockfile_prod`

This flow accepts the application source metadata and refreshes the package files used by the scans.

It can:

- accept uploaded or direct file URLs
- pull `package.json` and `package-lock.json` from a GitHub repository
- save `site_url`, `github_repo`, and related runtime values in Kestra KV Store

### 2. `security_scan_layer1_prod`

This flow performs dependency scanning using a Node-based Docker environment.

It scans the latest uploaded lockfile and package manifest and produces a structured JSON result that is passed back to the main flow.

### 3. `security_scan_layer2_prod`

This flow performs runtime security scanning using Playwright in Docker.

It opens the deployed site in a browser, watches network activity, compares seen domains against an allowlist, and returns a structured JSON result.

### 4. `security_pipeline_main_prod`

This is the main orchestration flow.

It:

- triggers on schedule
- triggers from webhook events such as deployment or PR merge pipelines
- refreshes the latest package files
- runs Layer 1 and Layer 2 one after another
- collects both outputs
- builds a final combined security summary
- sends one curated Slack message
- checks Vercel deployment status
- optionally rolls back to the previous safe deployment

## Trigger modes

This setup supports two trigger modes:

1. **Scheduled trigger**  
   Runs automatically at a fixed interval such as every 6 hours.

2. **Webhook trigger**  
   Runs when called by GitHub Actions, Vercel webhooks, or any CI/CD workflow after merge or deploy.

## Runtime state

Dynamic runtime values are stored in Kestra KV Store, including:

- `security/site_url`
- `security/github_repo`
- `security/github_branch`

This allows the pipeline to reuse the latest known app configuration between executions and keep checking the newest version of the package files without manual re-entry every time.

## Vercel rollback

If the final security summary determines that the deployment is unsafe and rollback is advised, the pipeline can:

1. fetch recent Vercel deployments
2. identify the previous safe deployment
3. reassign the production alias to that deployment

This gives the pipeline an automatic fallback path when a risky deployment reaches production.

## Typical flow

1. A developer merges a PR or a new deployment goes live.
2. A webhook or schedule starts the main Kestra flow.
3. The latest package files are refreshed.
4. Layer 1 scans dependencies.
5. Layer 2 checks runtime browser behavior.
6. Gemini creates one final combined security summary.
7. Slack receives one final alert.
8. If the result is critical, Vercel can be rolled back to the previous safe deployment.

## Running locally with Docker Compose

This repository includes a `docker-compose.yaml` file for running Kestra and PostgreSQL locally.

### 1. Create environment file

Copy `.env.example` to `.env` and fill in the required values.

```bash
cp .env.example .env
```

### 2. Start the stack

```bash
docker compose up -d
```

### 3. Open Kestra

Open:

```text
http://localhost:8080
```

### 4. Import the flows manually

This project does not rely on automatic local flow sync.

After Kestra starts:

1. Open the Kestra UI
2. Go to **Flows**
3. Create or import each flow manually
4. Copy and paste the YAML files from the `kestra/` folder:
   - `upload_lockfile_prod.yml`
   - `security_scan_layer1_prod.yml`
   - `security_scan_layer2_prod.yml`
   - `security_pipeline_main_prod.yml`

## Required secrets

Configure the following secrets in Kestra before running the production pipeline:

- `GEMINI_API_KEY`
- `SLACK_WEBHOOK_URL`
- `VERCEL_TOKEN`
- `PROD_PIPELINE_WEBHOOK_KEY`

Add this only if your upload flow uses authenticated GitHub API access:

- `GITHUB_TOKEN`

## Use cases

- detect malicious packages after PR merge
- detect suspicious runtime network behavior after deploy
- continue checking even after CI/CD completes successfully
- alert teams quickly in Slack when risky deployments reach production
- roll back unsafe frontend deployments on Vercel

## Security note

Do not hardcode credentials, tokens, Slack webhooks, or API keys directly in the flow YAML files or in `docker-compose.yaml`.

Store all secrets in Kestra Secrets or secure environment variables before pushing the project to GitHub.
