# How the docker skill was verified

Evidence behind the claims in [SKILL.md](../SKILL.md). Moved out of it so that it is not read on every load.

- Written from friction met on a Windows 11 host with Docker Desktop and Linux containers (first commit 2026-09-10, last change 2026-10-08). The Docker Desktop version was not recorded.
- Re-checked on 2026-10-29 with Docker Engine 28.0.1 and Compose v2.33.1 on Docker Desktop (Windows 11, Linux containers), with throwaway containers that were removed afterwards:
  - `docker compose up -d --build frontend-ui` exits 1 when the `depends_on` service `backend-api` has a failing build (`RUN false`); `docker compose build frontend-ui` then succeeds and `docker compose up -d --no-deps frontend-ui` starts only that container (section 1).
  - A host directory mounted over a path hides what the image had there (0 entries in `/usr/local/lib/node_modules`); an anonymous volume at that path inside a bind-mounted parent shows the image's own content again (`corepack`, `npm`) (section 4).
  - Without `MSYS_NO_PATHCONV=1`, Git Bash turns `/etc/hostname` into `C:/Program Files/Git/etc/hostname` (section 6).
  - `netsh interface ipv4 show excludedportrange protocol=tcp` lists 80, 8082 and 49252-49351 on this machine, and a `docker run -d --name x -p` that fails on a busy port leaves the container in state `Created` (section 6).
  - Borrowing a container's `mysql` client to reach another server works (section 3).
- The stale socket (section 5), partly: `userAnalyticsOtlpHttp.sock` is a zero-byte entry in `AppData\Local\Docker\run` that `dir` lists but that cannot be opened; `fsutil reparsepoint query` on it fails with `Error 1920: The file cannot be accessed by the system`, as the section says. Here Docker was running, so the file was the live socket and no start failure occurred. The failure itself and the restart fix need a crash and a restart of Windows and were not tried; no delete was attempted.
- The UTF-8 BOM item (section 2), partly: with `charset = utf-8-bom` in `.editorconfig`, `dotnet format whitespace --verify-no-changes` (SDK 10.0.401, on Windows) reports `error CHARSET: Fix file encoding` for a `.cs` file without a BOM, and with `charset = utf-8` it reports the same for a file that has one. That is the mechanism; EF migration files and a Linux container run were not tried.
- Not run on purpose: crashing Docker Desktop to recreate the stale-socket start failure (it may not start again until Windows is restarted) and pulling the .NET SDK image (about 0.9 GB) for the BOM check inside a Linux container were not approved on 2026-10-29, so both stay untested.
- Not re-run: the `docker pull` size note and the `docker tag` retag note.
