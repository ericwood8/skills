---
name: aws-lightsail-container-deploy
description: Hard-won operational knowledge for deploying multi-container apps to AWS Lightsail Container Service — container naming rules, image versioning, the real create-container-service-deployment API shape, pod-style shared networking, the deployment state model, and log-based debugging. Use when running `aws lightsail` container-service commands, deploying or redeploying to Lightsail, editing a Lightsail containers.json/public-endpoint.json, or debugging a Lightsail container-service error.
---

# AWS Lightsail Container Service Deploy

Load this before writing any `aws lightsail` container-service command or touching `containers.json` — the API and runtime model here diverge from both plain ECS and Docker Compose in ways that produce confusing errors if you guess.

## Quick start — the correct deploy sequence

```bash
# 1. Get the CURRENT highest version for every image — never assume/increment manually, never use ":latest"
aws lightsail get-container-images --service-name <service> --query "containerImages[].image"

# 2. Push a new image, note the printed version it lands on
aws lightsail push-container-image --service-name <service> --label <label> --image <local-image>:local

# 3. Update containers.json with the new numeric version for the image(s) that changed

# 4. Deploy — containers and publicEndpoint are SEPARATE flags, not one --cli-input-json payload
aws lightsail create-container-service-deployment \
  --service-name <service> \
  --containers file://containers.json \
  --public-endpoint file://public-endpoint.json

# 5. Poll the DEPLOYMENT state (not the service state) until ACTIVE
aws lightsail get-container-services --service-name <service> \
  --query "containerServices[0].currentDeployment.{version:version,state:state}"

# 6. On failure, pull real logs per-container rather than guessing
aws lightsail get-container-log --service-name <service> --container-name <name> --query "logEvents[-20:]"
```

## Core rules

1. **Container names**: hyphens only, no underscores — must match `^(?:[a-z0-9]{1,2}|[a-z0-9][a-z0-9-]+[a-z0-9])$`.
2. **Image versions are numeric-only** — there is no `:latest`/`.LATEST`. Always re-check `get-container-images` right before deploying.
3. **`create-container-service-deployment` takes `--containers file://...` and `--public-endpoint file://...`** as separate parameters — never `--cli-input-json` with the container map as the whole payload.
4. **No per-container DNS.** All containers in one deployment share a single network namespace (pod-style). Reach a sibling via `localhost:<port>`, never its container name — that only works in Docker Compose.
5. **Only one container per deployment can be public** (`publicEndpoint`). Multiple logically-public services (SPA + API) need a reverse-proxy container (nginx) as the sole public endpoint, routing internally over `localhost`. Side effect: same-origin, so no CORS needed.
6. **No `depends_on` equivalent** — containers start in parallel with no ordering guarantee. Anything that resolves a sibling hostname at startup (e.g. nginx `proxy_pass`) needs lazy/per-request resolution, not a literal target, or it can crash before a sibling is even up.
7. **Two different `state` fields**: the container *service's* state (`READY` — there is no "RUNNING") vs. an individual *deployment's* state (`ACTIVATING` → `ACTIVE`/`FAILED`, superseded ones go `INACTIVE`). `ACTIVATING` can legitimately sit for several minutes.
8. **No IaC path fits this well** — it's CLI-managed (`push-container-image` + `create-container-service-deployment`), not Terraform/CloudFormation-first for a small service like this.
9. **Secrets live in plain env vars inside `containers.json`** — there's no secrets-manager integration at this tier. Gitignore the real file; keep a redacted `.example` sibling checked in for shape.

Full symptom → cause → fix detail (exact error text as actually seen) for every rule above: see [REFERENCE.md](REFERENCE.md).
