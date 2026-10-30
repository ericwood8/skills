---
name: docker
description: Use when working with a Dockerfile or docker-compose.yml, or diagnosing a Docker build or runtime problem, especially on a Windows host running Linux containers. Trigger words - Docker Desktop will not start, stale socket, dependency rebuild, UTF-8 BOM, node_modules bind mount, docker from Git Bash, borrow a container's client.
---

# Docker

A handful of gotchas from real Docker/Compose friction on a Windows-host + Linux-container setup, not a Docker tutorial.

## 1. `--build` on one service still rebuilds its dependencies

**Symptom:** `docker compose up -d --build frontend-ui` fails on an unrelated, currently-broken `backend-api` build, although only the frontend was requested.

**Cause:** Compose builds/starts anything the target service `depends_on`, not just the named service.

**Fix:** build the target in isolation first — `docker compose build <service>` (build only, doesn't touch dependencies) — then restart just that container without its siblings: `docker compose up -d --no-deps <service>`.

## 2. UTF-8 BOM causes cross-platform friction

**Symptom:** a Windows-ecosystem `.editorconfig` mandating `charset = utf-8-bom` produces files that break tooling once it runs inside Linux containers — in this project it failed `dotnet format --verify-no-changes` on auto-generated EF migration files.

**Fix:** when a Windows-authored repo's editor defaults assume BOM but the containers are Linux-based, expect this friction and strip the BOM from the generated files rather than fighting the formatter or trying to make every container Windows-based.

## 3. Borrow a CLI client from a running container instead of installing one

**Symptom:** need to run a one-off CLI command (e.g. `mysql`) against some database, but the client isn't installed on the Windows host.

**Fix:** reuse an already-running local container's bundled client against any target, including a remote one: `docker exec <local-container> mysql -h <any-host> -u <user> -p'<password>' <db> -e "<sql>"`. Works equally well against the local dev DB and a remote database (e.g. AWS RDS/Lightsail) — no need to install anything extra on the host.

## 4. Bind-mounting source on Windows can shadow the container's own `node_modules`

**Symptom:** a Node container built fine but breaks once a source bind mount is added — packages built for Linux get overwritten by whatever (or nothing) is on the Windows host.

**Cause:** a bind mount like `./frontend:/app` overwrites the container's own `node_modules` (installed for Linux) with the host's copy.

**Fix:** add an anonymous volume for just that path so the container keeps its own copy: `volumes: [./frontend:/app, /app/node_modules]`.

## 5. Docker Desktop will not start: "removing stale socket ... userAnalyticsOtlpHttp.sock: The file cannot be accessed by the system"

**Symptom:** after a crash or forced quit, Docker Desktop shows `running OTel manager: removing stale socket: remove C:\Users\<you>\AppData\Local\Docker\run\userAnalyticsOtlpHttp.sock` and the engine pipe (`dockerDesktopLinuxEngine`) never appears.

**Cause:** the leftover is a Unix-socket stub (a reparse point) that Windows reports as error 1920 for every operation. `del /f`, `Remove-Item -Force`, `fsutil reparsepoint delete` and `[System.IO.File]::Delete('\\?\...')` all fail with the same error, even with Docker fully quit.

**Fix:** quit Docker Desktop, restart Windows, then start Docker Desktop. The file is gone after the restart. If it is not, delete it in File Explorer. Do not spend time on more delete attempts.

## 6. Running docker from Git Bash on Windows

- `export MSYS_NO_PATHCONV=1` first, or Git Bash rewrites container paths such as `/etc/nginx/...` into `C:/Program Files/Git/etc/...`.
- Mount host files with `-v "$(cygpath -w "$PWD/file")":/path/in/container:ro`.
- A failed `docker run -p` can still leave the named container created; `docker rm -f name` before reusing the name.
- Docker Desktop's engine has stopped by itself in the middle of a long session once (not reproduced): run the checks that need a throwaway container in one sitting and confirm `docker ps` before relying on it.
- Windows reserves port ranges (`netsh interface ipv4 show excludedportrange protocol=tcp`); a port in one fails with "forbidden by its access permissions". Use ports above 18000 for throwaway tests.
- `docker pull` and a started Docker Desktop are downloads/starts the user should know about; say which image and roughly how big before pulling.
- Retag an image under a local name for a registry push with a plain `docker tag <image>:<version> <local-name>:local`; no rebuild (for example `mcr.microsoft.com/mssql/server`).
- For nginx configs see the `nginx-config-testing` skill.

## How this was verified

Tested on this machine. The evidence, the versions used and the items **not re-run** are in [reference/verification.md](reference/verification.md): read it before saying a claim here was tested, and treat anything listed there as not re-run as unverified.

## What this does not prove

- That a rule holds on a Linux or macOS host, with Windows containers, Podman, or a later Docker Desktop. The stale-socket fix (restart Windows) may not be needed in a newer version.
- That a Compose behavior (`--build` rebuilding dependencies) holds for every Compose version.
