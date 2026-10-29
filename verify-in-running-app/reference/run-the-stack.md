# Run the stack

Read this when: you start the API and front end for a click-through, stop them again, or build a sample that has never compiled a template.

- **API:** set `ConnectionStrings__<Name>` (and any `Aspire__...__ConnectionString`) in the same command, then `dotnet run --project <Api> --no-launch-profile --urls http://localhost:5084` in the background (a `Urls` entry in `appsettings.json` otherwise wins). A first admin on an empty user table needs `Bootstrap__AdminUserName` / `Bootstrap__AdminPassword` / `Jwt__Key`. Confirm with `curl` that a GET returns rows from the copy.
- **UI:** `ng serve --port 4200` in the background; export the API address variable a proxy config reads (`services__<name>__http__0=http://localhost:5084`) first. `curl http://localhost:4200/api/<route>` returning the same JSON proves the proxy.
- **Rebuild only after stopping the API** (`MSB3027` file lock). `dotnet run` starts a child process: stop with `taskkill /PID <id> /T /F`.
- **Stop processes by listening port** (`netstat -ano | grep ":5084 .*LISTENING"` or `Get-NetTCPConnection`), printing the pid's path first. Filtering `Get-CimInstance Win32_Process` by command line also matches the shell running the command and kills it. Stop only what this session started and say which URLs are up.
- **A sample never compiled with a template hides template bugs:** build it once. Copy the sample without `bin`/`obj` to the scratchpad, drop the generated files in, build, run on a spare port (`--urls http://localhost:5091`), curl it, delete the copy. For a front-end page add a temporary route (`App.tsx` / `app.routes.ts`) after backing up the route file, run `npx tsc -b` or `npx ng build` and the page's vitest file, then revert.
- Another database, same front end: stop the other API on that port and start the new one there (React 5080, Angular 5081); the front end reads it unchanged. Say which API owns the port.
