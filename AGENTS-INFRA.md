# Infrastructure Deployment Guide

> How to deploy applications to Volley's Kubernetes infrastructure.
> Read this when working on Docker, CI/CD, deployment, infrastructure, or Kubernetes tasks.

---

## Overview

Volley runs applications on an **AWS EKS (Kubernetes) cluster** managed via **Flux GitOps**. Deploying an application requires:

1. A runnable **Docker image** pushed to ECR
2. **IAM roles** for AWS resource access
3. **Helm releases** defining the K8s deployment
4. **Flux configs** for automated deployments

---

## Key Repositories

| Repository | Purpose | When to use |
|-----------|---------|-------------|
| [volley-infra](https://github.com/Volley-Inc/volley-infra) | AWS resources via Terraform (Atlantis). IAM roles, Redis clusters, ECR repos, etc. | Creating AWS resources, adjusting IAM permissions |
| [volley-infra-tenants](https://github.com/Volley-Inc/volley-infra-tenants) | Helm releases and K8s manifests for all deployed apps | Configuring app deployments (env vars, resources, scaling, secrets, hostnames) |
| [kubernetes](https://github.com/Volley-Inc/kubernetes) | Flux configs, namespace creation, RBAC, alerting | Onboarding new apps, setting up CD automation |
| [helm-charts](https://github.com/Volley-Inc/helm-charts) | Shared Helm charts (e.g. `app` chart at `charts/app/0.1.1`) | Reference only — used by helm releases |

---

## Docker Image Requirements

### Dockerfile Pattern (VGF monorepo)

All VGF game servers follow this multi-stage pattern:

```
Stage 1: base        → node:22-slim + pnpm (corepack enable)
Stage 2: dependencies → Copy package.json files + pnpm install --frozen-lockfile
Stage 3: build       → Copy source + build all packages + pnpm deploy --prod
Stage 4: production  → Copy built artifacts into minimal runtime image
```

**Critical details:**
- Use `--mount=type=secret,id=npm_token` for `@volley` private package auth during install
- Use `pnpm deploy --filter=<package> --prod /prod/<name>` to create a pruned production bundle
- Assets (puzzle data, audio, etc.) must be copied to match the path the server resolves at runtime
- The server entry point is `node dist/index.js` (compiled TypeScript)
- Expose port **8080** (production default)

### `.npmrc` for Private Packages

Required at project root for Docker builds to access `@volley/*` packages:

```
//registry.npmjs.org/:_authToken=${NPM_TOKEN}
```

The `NPM_TOKEN` is passed as a Docker build secret, **not** a build arg (secrets don't leak into image layers).

### ECR Registry

All images are pushed to: `375633680607.dkr.ecr.us-east-1.amazonaws.com`

Image names follow the pattern: `375633680607.dkr.ecr.us-east-1.amazonaws.com/<app-name>:<tag>`

Tags follow: `<branch>-<short-sha>-<timestamp>`

---

## CI/CD Workflow

### GitHub Actions CD (`.github/workflows/cd.yml`)

Triggered on push to `main`. Steps:
1. Checkout code
2. Login to ECR using `AWS_ACCESS_KEY` / `AWS_SECRET_ACCESS_KEY` secrets
3. Build Docker image with `NPM_TOKEN` secret
4. Push to ECR with tag `main-<sha>-<timestamp>`

**Required GitHub Secrets:**
- `AWS_ACCESS_KEY` — ECR push access
- `AWS_SECRET_ACCESS_KEY` — ECR push access
- `NPM_TOKEN` — npmjs.org token for `@volley` private packages

### Flux Automatic Image Updates

Once configured in the [kubernetes](https://github.com/Volley-Inc/kubernetes) repo, Flux watches for new images in ECR and automatically updates the helm release in volley-infra-tenants, triggering a rolling deployment.

---

## Onboarding a New Application (Step by Step)

### Step 1: Create ECR Repository

**Repo:** `volley-infra` (Terraform PR)

Create an ECR repository for storing Docker images. Submit a PR to the terraform directory.

### Step 2: Create IAM Roles

**Repo:** `volley-infra` (Terraform PR)

Create IAM roles for dev, staging, and prod environments. These roles use OIDC to allow the EKS cluster's service accounts to assume them. Attach policies for any AWS resources the app needs (S3, Lambda, etc.).

### Step 3: Add Helm Release Configs

**Repo:** `volley-infra-tenants`

Add helm release files for each environment (dev, staging, prod). These configure:
- **Hostname** (internal or public)
- **Resource requests/limits** (CPU, memory)
- **Autoscaling** (min/max replicas, target CPU)
- **Environment variables**
- **Secrets** (via SecretProviderClass from AWS Secrets Manager)
- **Service account annotations** (IAM role ARN)

Example structure:
```
<app-name>/kubernetes/
├── dev/
│   ├── config.yaml          # Helm release values
│   └── secret-provider-class.yaml
├── staging/
│   ├── config.yaml
│   └── secret-provider-class.yaml
└── prod/
    ├── config.yaml
    └── secret-provider-class.yaml
```

### Step 4: Create Flux Configs and Namespaces

**Repo:** `kubernetes`

Run the provided scripts to generate:

| File | Purpose |
|------|---------|
| `<app>-image-auto-update.yaml` | Flux automatic image update config (CD trigger) |
| `sync.yaml` | Flux kustomization — what to deploy in each namespace |
| `rbac.yaml` | RBAC for developer access to namespaces |
| `alertmanagerconfig.yaml` | Prometheus alerting for the namespaces |
| `slack-<app>-notifications.yaml` | Flux notification provider for Slack |
| `notification-info.yaml` | Flux alert config per namespace |

---

## Environment Variables (This Project)

| Variable | Default | Required | Purpose |
|----------|---------|----------|---------|
| `NODE_ENV` | `production` | No | Runtime environment |
| `PORT` | `8080` | No | Server port |
| `REDIS_URL` | `redis://localhost:6379` | Yes (prod) | Redis connection |
| `LOG_LEVEL` | `info` | No | Logging verbosity |
| `CORS_ORIGIN` | `https://play.volley.tv` | No | Allowed CORS origins (comma-separated) |
| `SHUTDOWN_TIMEOUT` | `25000` | No | Graceful shutdown timeout (ms) |
| `STAGE` | `production` | No | Deployment stage |
| `DATABASE_URL` | — | No | Database connection string |
| `AMPLITUDE_API_KEY` | — | No | Analytics |
| `SEGMENT_WRITE_KEY` | — | No | Analytics |
| `CODE_MAP_CREATE_URL` | `https://auth.volley.tv/code-map` | No | Room code service |
| `CODE_MAP_RESOLVE_URL` | `https://auth.volley.tv/code-map` | No | Room code service |
| `DD_ENV` | — | No | Enables Datadog APM tracing when set |

---

## Common Tasks

### Updating environment variables
Submit a PR to `volley-infra-tenants` modifying the relevant `config.yaml`.

### Adjusting resource limits or scaling
Submit a PR to `volley-infra-tenants` modifying `resources` or `autoscaling` in `config.yaml`.

### Adjusting IAM permissions
Submit a PR to `volley-infra` modifying the Terraform IAM policy attachments.

### Adding secrets
1. Store the secret in AWS Secrets Manager
2. Add a `SecretProviderClass` entry in `volley-infra-tenants`
3. Reference it as an env var in the helm release `config.yaml`

### Getting help
Post in **#infra-support** Slack channel.

---

## Keyword Triggers

When a task involves any of these keywords, read this document:
`docker`, `dockerfile`, `container`, `kubernetes`, `k8s`, `deploy`, `deployment`, `ecr`, `helm`, `flux`, `infrastructure`, `infra`, `iam`, `eks`, `ci/cd`, `cd.yml`, `production build`, `docker image`
