# Gym Platform GitOps

This repository is the single source of truth for what runs on the gym
platform Kubernetes (EKS) cluster. It holds Argo CD `Application` objects and
Kustomize manifests for every microservice, and Argo CD continuously
reconciles the cluster to match what is committed here. Nothing is deployed
with `kubectl apply` by hand in normal operation — you change YAML, open a
pull request, merge, and Argo CD rolls the change out.

The cluster itself (VPC, EKS, ECR, RDS, IAM/IRSA, Jenkins, Argo CD, External
Secrets) is provisioned separately by the `gym-platform-terraform` repo. This
repo assumes that cluster already exists.

## Overview

- **GitOps engine:** Argo CD, running in the `argocd` namespace, using an
  "App of Apps" (root application) pattern.
- **Templating:** Kustomize, with a shared `base/` per service and one
  `overlays/<env>/` per environment.
- **Environments:** `dev` (namespace `gym-dev`) and `prod` (namespace
  `gym-prod`).
- **Image delivery:** application images are built and pushed to Amazon ECR by
  per-service Jenkins jobs. Argo CD Image Updater watches ECR and writes new
  image tags back into the overlay `kustomization.yaml` files in Git.
- **Secrets:** no secret values live in this repo. Every `Secret` is an
  `ExternalSecret` that External Secrets Operator hydrates from AWS Secrets
  Manager (`gym/dev/*` keys) via the `aws-secrets-store` `ClusterSecretStore`.
- **Ingress:** a single ALB `Ingress` on the api-gateway routes API path
  prefixes to the gateway and everything else to the frontend SPA. TLS is
  issued by cert-manager (Let's Encrypt / DuckDNS) and mirrored into AWS ACM
  by the orchestrator pipeline.
- **Orchestration:** a top-level `Jenkinsfile` bootstraps the Argo CD project,
  root app, and Image Updater config, triggers the per-service build jobs,
  syncs the TLS cert to ACM, updates DuckDNS, and runs a smoke test.

## Repo Structure

- `argocd/` — Argo CD control-plane objects
  - `projects/gym-project.yaml` — the `gym-platform` `AppProject`; whitelists
    this repo as the only allowed source and `argocd` / `gym-dev` /
    `gym-prod` as the only allowed destinations
  - `root-apps/root-app-dev.yaml` — root "App of Apps" for dev; scans
    `services/` on the `env/staging` branch and adopts every
    `application-dev.yaml` it finds
  - `root-apps/root-app-prod.yaml` — root app for prod (tracks `main`);
    currently a stub, see [Environments and Branching](#environments-and-branching)

- `services/<service>/` — one directory per microservice, all following the
  same layout
  - `application-dev.yaml` — Argo CD `Application` for dev; points at
    `overlays/dev`, branch `env/staging`, namespace `gym-dev`, auto-sync with
    prune + self-heal
  - `application-prod.yaml` — Argo CD `Application` for prod; points at
    `overlays/prod`, branch `main`, namespace `gym-prod`, auto-sync with
    prune, self-heal off
  - `base/` — environment-agnostic manifests (Deployment, Service, ConfigMap,
    `ExternalSecret`, and — where the service owns its datastore — a
    StatefulSet plus headless Service and a one-shot migration Job)
  - `overlays/dev/` and `overlays/prod/` — a `kustomization.yaml` that
    references `../../base`, sets the namespace, declares the `images:` entry
    that Image Updater rewrites, and applies `patch-deployment.yaml`
    (replica count and resource requests/limits per environment)

- `shared/` — cluster-wide resources applied once by the orchestrator
  - `dev-namespace.yaml`, `prod-namespace.yaml` — the `gym-dev` / `gym-prod`
    namespaces
  - `image-updater.yaml` — Argo CD Image Updater `ImageUpdater` config; one
    entry per buildable service mapping an ECR repo to its overlay
    `kustomization.yaml`. `${ECR_REGISTRY}` and `${ENVIRONMENT}` are filled in
    by `envsubst` in the pipeline before it is applied.

- `Jenkinsfile` — the orchestrator pipeline (see [Quick Start](#quick-start-deploy-with-jenkins-recommended))

## Services

| Service | ECR image | Bundled datastore | Notes |
|---|---|---|---|
| api-gateway | `gym-api-gateway` | — | Owns the ALB `Ingress` and the `api-gateway` ServiceAccount (S3 presigned uploads via EKS Pod Identity) |
| auth-service | `gym-auth-service` | Postgres StatefulSet | Migration Job applies `users` schema |
| profile-service | `gym-profile-service` | Postgres StatefulSet | |
| progress-service | `gym-progress-service` | MongoDB StatefulSet | |
| social-service | `gym-social-service` | Postgres StatefulSet | |
| catalog-service | `gym-catalog-service` | Postgres StatefulSet | |
| order-service | `gym-order-service` | Postgres StatefulSet | |
| payment-service | `gym-payment-service` | Postgres StatefulSet | |
| operations-service | `gym-operations-service` | Postgres StatefulSet + Redis | |
| ai-service | `gym-ai-service` | Postgres StatefulSet | LLM gateway token from Secrets Manager |
| frontend-service | `frontend-service` | — | Nginx serving the React SPA |
| kafka-service | *(upstream `apache/kafka`)* | PVC | Not built by Jenkins; deployed straight from manifests. Includes Kafka UI. Service is named `kafka`. |

## How It Works

```
 developer                Jenkins (orchestrator +          Argo CD                 EKS
 ─────────                 per-service build jobs)          ───────                 ───
     │                              │                          │                     │
     │ 1. edit manifests / merge PR │                          │                     │
     ├─────────────────────────────>│                          │                     │
     │                              │ 2. build+push image ─────┼──> Amazon ECR       │
     │                              │ 3. apply shared/,        │                     │
     │                              │    argocd/ project +     │                     │
     │                              │    root-app + image-updater ───────────────────┤
     │                              │                          │ 4. reconcile        │
     │                              │                          ├────────────────────>│
     │                              │                          │  (kustomize build,  │
     │                              │                          │   apply, self-heal) │
     │        Argo CD Image Updater │<── 5. new ECR tag ───────>│                     │
     │        commits new tag to Git│                          │ 6. reconcile again  │
     │                              │                          ├────────────────────>│
```

1. You change YAML in `services/<service>/...` and merge it to the branch the
   target environment tracks.
2. The orchestrator pipeline (or a manual run of a per-service job) builds the
   container image and pushes it to ECR.
3. The orchestrator applies the namespace, the `gym-platform` `AppProject`,
   the environment root app, and the Image Updater config.
4. The root app discovers each `application-<env>.yaml` and creates the child
   Argo CD `Application`s; Argo CD runs `kustomize build` on each overlay and
   applies the result, then keeps the cluster reconciled (auto-sync +
   self-heal in dev).
5. Argo CD Image Updater notices a newer image tag in ECR
   (`updateStrategy: newest-build`) and commits the new tag into the overlay
   `kustomization.yaml`.
6. That commit triggers another reconcile, rolling the new image out.

## Environments and Branching

| | Dev | Prod |
|---|---|---|
| Namespace | `gym-dev` | `gym-prod` |
| Branch tracked by Argo CD | `env/staging` | `main` |
| Overlay | `overlays/dev` | `overlays/prod` |
| Sync policy | automated, prune + self-heal | automated, prune, **no** self-heal |
| Replicas (example: auth-service) | 1 | 3 |

- **`env/staging`** is the integration branch. All `application-dev.yaml`
  files and `root-app-dev.yaml` point here, so merging to `env/staging`
  deploys to `gym-dev`.
- **`main`** is the promotion branch for `gym-prod`. Promote by merging
  `env/staging` into `main`.
- `root-app-prod.yaml` is currently a placeholder — with Kustomize overlays
  the prod root app needs to enumerate the per-service `application-prod.yaml`
  objects the same way the dev root app does. Finish this before relying on
  prod auto-sync.
- Argo CD Image Updater's `writeBackConfig` commits tag bumps to `main`.
  Keep this in mind when reconciling `env/staging` with `main`.

## Prerequisites

- An EKS cluster provisioned by `gym-platform-terraform`, with:
  - Argo CD installed in the `argocd` namespace
  - External Secrets Operator and the `aws-secrets-store` `ClusterSecretStore`
  - AWS Load Balancer Controller (for the ALB `Ingress`)
  - cert-manager with the `letsencrypt-duckdns` `ClusterIssuer`
  - Argo CD Image Updater
- Application secrets seeded in AWS Secrets Manager under `gym/dev/*`
  (`gym-platform-terraform/scripts/seed-app-secrets.sh`).
- Jenkins with AWS credentials (`aws-access-key-id`, `aws-secret-access-key`),
  a `duckdns-token` secret credential, and one build job per buildable
  service (job names match the ECR image names, e.g. `gym-auth-service`).
- Local tooling for manual work: `kubectl`, `kustomize` (or `kubectl -k`),
  `argocd` CLI, `aws` CLI.

## Quick Start: Deploy with Jenkins (Recommended)

1. Create a Jenkins pipeline job pointing at this repo with **Script Path**
   `Jenkinsfile`.
2. Build with parameters:
   - `ENVIRONMENT` — `dev` or `prod` (selects `shared/<env>-namespace.yaml`,
     the `overlays/<env>` path, and the `gym-platform-root-<env>` app)
   - `SKIP_BUILD` — `true` to skip the per-service build jobs and only
     (re)apply manifests
   - `SYNC_ARGOCD` — `true` to trigger a root-app sync and wait for every
     child app to report Synced + Healthy
3. The pipeline will, in order:
   - bootstrap `aws`, `kubectl`, and `envsubst` into the workspace
   - `aws eks update-kubeconfig` for `gym-cluster` and resolve the ECR
     registry from the caller identity
   - apply `shared/<env>-namespace.yaml` and the `envsubst`-rendered
     `shared/image-updater.yaml`
   - trigger each per-service build/push job (unless `SKIP_BUILD`)
   - apply `argocd/projects/gym-project.yaml` and
     `argocd/root-apps/root-app-<env>.yaml`
   - wait for cert-manager to produce the `api-gateway-tls` secret, then
     import/update it in AWS ACM and annotate the `Ingress` with the cert ARN
   - sync the root app and wait for all child apps to become Healthy
   - print the Argo CD admin password and port-forward instructions
   - resolve the ALB hostname, update the DuckDNS `A` record, and smoke-test
     `https://iti-gym-platform.duckdns.org/health` and `/auth/register`

## Manual Bootstrap (Without Jenkins)

Run from the repo root with `kubectl` already pointed at the cluster:

```bash
# 1. Namespace
kubectl apply -f shared/dev-namespace.yaml

# 2. Argo CD Image Updater config (fill in the placeholders first)
export ECR_REGISTRY="<account-id>.dkr.ecr.us-east-1.amazonaws.com"
export ENVIRONMENT="dev"
envsubst < shared/image-updater.yaml | kubectl apply -f -

# 3. Argo CD project and the dev root app
kubectl apply -f argocd/projects/gym-project.yaml
kubectl apply -f argocd/root-apps/root-app-dev.yaml

# 4. Watch the child apps appear and sync
kubectl get applications -n argocd -w
```

To render a single service's manifests locally without applying:

```bash
kubectl kustomize services/auth-service/overlays/dev
```

## Common Commands

```bash
# List all Argo CD apps and their sync/health status
kubectl get applications -n argocd

# Inspect one app
kubectl describe application auth-service-dev -n argocd

# Force a sync of the whole environment via the root app
kubectl patch application gym-platform-root-dev -n argocd --type merge \
  -p '{"operation":{"sync":{"prune":true}}}'

# Or with the argocd CLI
argocd app sync gym-platform-root-dev
argocd app list

# Diff local manifests against the live cluster
kubectl diff -k services/auth-service/overlays/dev

# Pods and logs for a service
kubectl get pods -n gym-dev
kubectl logs deployment/auth-service -n gym-dev
```

## Accessing Deployed Services

### Public endpoint

`https://iti-gym-platform.duckdns.org` — the ALB `Ingress`. API prefixes
(`/auth`, `/api`, `/orders`, `/payments`, `/progress`, `/social`, `/chat`,
`/commerce`, `/ai`, `/operations`, `/webhooks`, `/uploads`, `/health`) route
to the api-gateway; everything else serves the React SPA from
frontend-service.

### Argo CD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
# https://localhost:8081  (user: admin)
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

### Kafka UI

Exposed via its own `Ingress` in `services/kafka-service/base/kafka-ui/`.

## Troubleshooting

### Application stuck `OutOfSync` or `Missing`

```bash
kubectl describe application <svc>-dev -n argocd
```

Check that the branch the `Application` tracks (`env/staging` for dev)
actually contains your change, and that `kustomize build` on the overlay
succeeds locally.

### `SharedResourceWarning` / resource claimed by another app

Two `Application`s are managing the same object. Usually the root app and a
child app both matched a manifest — tighten the root app's `directory.include`
glob or remove the duplicate.

### ExternalSecret not becoming `Ready`

```bash
kubectl describe externalsecret <name> -n gym-dev
kubectl get clustersecretstore aws-secrets-store
```

The referenced key must exist in AWS Secrets Manager (`gym/dev/...`) and the
cluster's IRSA role must allow `secretsmanager:GetSecretValue`. Re-run the
seed script in `gym-platform-terraform` if a key is missing.

### Postgres pod `CrashLoopBackOff` on first deploy

Often a stale PVC from an earlier attempt — Postgres only runs its
`POSTGRES_DB` init on an empty data directory. Delete the StatefulSet and its
`volumeClaimTemplates` PVC (`postgres-storage-<svc>-postgres-0`) and let Argo
CD recreate them (this wipes that environment's data).

### TLS / ACM

If HTTPS fails, confirm cert-manager issued `api-gateway-tls` in the target
namespace and that the `Ingress` has an
`alb.ingress.kubernetes.io/certificate-arn` annotation. The orchestrator
pipeline's "Sync TLS Certificate to AWS ACM" stage sets this; re-run it if the
cert was rotated.

### Image not updating

```bash
kubectl logs -n argocd deployment/argocd-image-updater
```

Check that `shared/image-updater.yaml` was applied with `${ECR_REGISTRY}` /
`${ENVIRONMENT}` substituted, and that Image Updater can push to `main`.

## Notes

- This repo contains **no** secret values — only `ExternalSecret` references.
- `kubectl apply` by hand is for bootstrap and emergencies only. In steady
  state, change Git and let Argo CD reconcile; manual changes in dev are
  reverted by self-heal.
- The `images:` entries in the overlay `kustomization.yaml` files are
  intentionally left without a tag in Git — Argo CD Image Updater fills them
  in.
- `kafka-service` is deployed from upstream images and is not part of the
  Jenkins build matrix.
- `gym-cluster`, region `us-east-1`, and the `iti-gym-platform` DuckDNS domain
  are hard-coded in the `Jenkinsfile` `environment` block.

## Goals

This repository should let an operator:

- See exactly what is deployed to every environment by reading Git.
- Deploy a service by adding a `base/` + `overlays/` directory and an
  `application-<env>.yaml`, with no change to the root app.
- Promote dev to prod by merging `env/staging` into `main`.
- Roll back by reverting a commit.
- Keep all credentials in AWS Secrets Manager, never in a manifest.

To onboard a new service, copy an existing `services/<service>/` directory,
adjust the manifests and image name, add an entry to
`shared/image-updater.yaml`, and create the matching Jenkins build job.
