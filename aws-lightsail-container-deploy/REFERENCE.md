# AWS Lightsail Container Service — Gotcha Reference

Each entry: the symptom as actually seen, why it happens, and the fix. All verified firsthand (Critical Viewer project, a 3-container .NET/React app: nginx proxy + ASP.NET backend + React frontend on Lightsail Container Service, `nano` power, `us-east-2`).

## 1. Container naming — underscores are rejected

**Symptom:**
```
An error occurred (InvalidInputException) when calling the CreateContainerServiceDeployment operation:
Container name "backend_api" does not match pattern: ^(?:[a-z0-9]{1,2}|[a-z0-9][a-z0-9-]+[a-z0-9])$
```

**Cause:** Lightsail container names allow lowercase alphanumerics and hyphens only — no underscores, unlike a lot of Docker-world naming conventions.

**Fix:** Use hyphens everywhere a container is named — `backend-api`, `frontend-ui`, not `backend_api`/`frontend_ui`. Keep this consistent with `containers.json` keys and any Dockerfile/compose service names you want to visually match.

## 2. Image versions are numeric-only

**Symptom:**
```
An error occurred (InvalidInputException) when calling the CreateContainerServiceDeployment operation:
Container "frontend-ui" has invalid image ":container-service-1.my-react-ui.LATEST".
Example of valid image: ":container-service-1.frontend-ui.123".
```

**Cause:** There is no `:latest`/`.LATEST` moving tag for Lightsail container images — only literal, monotonically-increasing integers per label, assigned by `push-container-image`.

**Fix:** Before every deployment, run:
```bash
aws lightsail get-container-images --service-name <service> --query "containerImages[].image"
```
and read off the actual current highest version per label. Never hardcode "latest" and never manually increment a remembered number — old versions can be deleted independently, so the true current version isn't always "last one you pushed + 1".

## 3. `create-container-service-deployment` payload shape

**Symptom:**
```
An error occurred (ParamValidation): Parameter validation failed:
Unknown parameter in input: "proxy", must be one of: serviceName, containers, publicEndpoint
Unknown parameter in input: "backend-api", must be one of: serviceName, containers, publicEndpoint
Unknown parameter in input: "frontend-ui", must be one of: serviceName, containers, publicEndpoint
```

**Cause:** This happens from running:
```bash
aws lightsail create-container-service-deployment --service-name <service> --cli-input-json file://containers.json
```
`--cli-input-json` expects the *entire* request payload — `{serviceName, containers, publicEndpoint}` — but a typical `containers.json` (as most guides and this project's own `deploy/lightsail/containers.json` are written) is just the container-name → container-spec map, which is only the *value* of the `containers` parameter, not the whole request.

**Fix:** Pass `--containers` and `--public-endpoint` as their own flags, each pointed at its own file:
```bash
aws lightsail create-container-service-deployment \
  --service-name <service> \
  --containers file://deploy/lightsail/containers.json \
  --public-endpoint file://deploy/lightsail/public-endpoint.json
```
where `public-endpoint.json` looks like:
```json
{
  "containerName": "proxy",
  "containerPort": 80,
  "healthCheck": {
    "path": "/api/health",
    "successCodes": "200",
    "intervalSeconds": 10,
    "timeoutSeconds": 5,
    "healthyThreshold": 2,
    "unhealthyThreshold": 5
  }
}
```

## 4. Pod-style shared networking — no per-container DNS

**Symptom (from a proxy container's logs):**
```
[error] 38#38: *9 backend-api could not be resolved (3: Host not found), client: ..., request: "GET /api/health HTTP/1.1", ...
```
paired with `502` responses at the public endpoint.

**Cause:** Docker Compose gives every service its own per-service DNS name on a shared bridge network (`backend-api` resolves from `proxy`). Lightsail Container Service does **not** work this way — every container in one deployment runs inside a single shared network namespace, pod-style. There is no per-container DNS name at all.

**Fix:** Address sibling containers via `localhost:<port>`, not their container name. If the same config needs to work in both Docker Compose (local) and Lightsail (real), parameterize the upstream host via an environment variable rather than hardcoding either value:
- Compose: `BACKEND_API_UPSTREAM=backend-api:8080`
- Lightsail: `BACKEND_API_UPSTREAM=localhost:8080`

## 5. Only one container can be public

**Symptom:** No error exactly — it's a design constraint. `publicEndpoint` accepts exactly one `containerName`.

**Cause:** Lightsail Container Service allows exactly one publicly-reachable container per deployment. An app shaped as "SPA calls a separate API" doesn't fit that directly.

**Fix:** Add a reverse-proxy container (nginx is the natural choice) as the sole public container, routing internally by path (`/api/*` → backend, everything else → frontend) over `localhost`. Bonus: this makes frontend and API same-origin from the browser's perspective, so CORS isn't needed for real traffic.

## 6. No `depends_on` — containers start in parallel

**Symptom:** A proxy container crashes outright at boot (not just individual requests failing) if `proxy_pass` targets a literal hostname that isn't resolvable yet.

**Cause:** Lightsail has no equivalent of Compose's `depends_on` to sequence container startup. All containers in a deployment start in parallel with no ordering guarantee.

**Fix (nginx-specific pattern that held up):** Use a `resolver` directive plus a `set`-based indirection so nginx resolves the upstream lazily, per-request, instead of once at startup:
```nginx
resolver ${NGINX_LOCAL_RESOLVERS} valid=10s;

location /api/ {
    set $backend_api ${BACKEND_API_UPSTREAM};
    proxy_pass http://$backend_api;
    ...
}
```
A bare literal `proxy_pass http://backend-api:8080;` fails hard at startup if resolution fails even once; the `set $var; proxy_pass http://$var;` form doesn't. `${NGINX_LOCAL_RESOLVERS}` comes from the official nginx image's own `15-local-resolvers.envsh` entrypoint step (reads `/etc/resolv.conf`), gated behind `NGINX_ENTRYPOINT_LOCAL_RESOLVERS=1` on the container — without that env var the script silently no-ops and the template substitution never happens.

## 7. Two different `state` fields

**Symptom:** Confusion between `"state": "READY"` on the service and `"state": "ACTIVATING"`/`"ACTIVE"`/`"FAILED"`/`"INACTIVE"` on deployments — there is no `"RUNNING"` state anywhere in this API, despite it being an intuitive guess.

**Cause:** `get-container-services` returns the container *service's* own state (`READY` when the service itself is up and serving *some* deployment) separately from `currentDeployment.state` / `nextDeployment.state`, which track the specific deployment version's rollout.

**Fix:** When waiting for a new deploy to take effect, poll the *deployment* state, not the service state:
```bash
aws lightsail get-container-services --service-name <service> \
  --query "containerServices[0].currentDeployment.{version:version,state:state}"
```
Expect `ACTIVATING` to persist for several minutes — that's normal, not stuck. A superseded previous deployment shows `INACTIVE` once the new one goes `ACTIVE`.

## 8. Debugging a failed/unhealthy deployment

**Command:**
```bash
aws lightsail get-container-log --service-name <service> --container-name <name> --query "logEvents[-20:]"
```

**Signatures actually seen and what they meant:**
- `"[deployment:4] Took too long"` in the app container's own logs — the app didn't bind/respond inside the health-check window in time. This is an app-level startup problem (e.g. slow migrations, misconfigured connection string causing a hang), not a Lightsail platform issue.
- `"<name> could not be resolved (3: Host not found)"` in the proxy's logs, paired with `502` responses — the pod-networking issue from item 4 (proxy using a container name instead of `localhost`).

## 9. No IaC path fits this tier well

Lightsail Container Service at this scale (a handful of containers, `nano`/`micro` power) doesn't have a natural Terraform/CloudFormation-first workflow the way ECS+RDS does — it's realistically CLI-managed: `push-container-image` + `create-container-service-deployment` by hand or from a script. Treat `containers.json` + `public-endpoint.json` as the actual source of truth for what's deployed, not as a rendering step from IaC.

## 10. Secrets management

There's no first-class secrets-manager integration for container environment variables at this tier — DB passwords, signing keys, etc. go in as plain env var values inside `containers.json`. Treat that file as sensitive:
- Gitignore the real `containers.json`.
- Keep a redacted `containers.json.example` checked in (same shape, placeholder values) so the structure is documented without leaking secrets.
