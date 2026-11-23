---
name: dotnet-major-upgrade
description: Use when moving an ASP.NET, EF Core and Aspire solution to a newer .NET major (for example net9.0 to net11.0), or when upgrading Aspire, Swashbuckle, Microsoft.OpenApi, OpenTelemetry or MSTest packages that come with it. Trigger words - TargetFramework, EF Core 11, ASPIRE012, AddNpmApp, AddJavaScriptApp, Missing script dev, Aspire.Hosting.NodeJs, OpenApiSecurityScheme, Microsoft.OpenApi.Models, MSTest 4, VSTest target is no longer supported, global.json test runner, NU1903.
---

# Moving a .NET solution to a new major (framework, EF Core, Aspire, Swashbuckle, MSTest)

Worked through on a .NET 9 Aspire solution (API, AppHost, ServiceDefaults, Common, CLI, MSTest project) moved to .NET 11, EF Core 11, Aspire 17, Swashbuckle 11 and MSTest 4. Every step ended with the whole solution built and its tests passing.

## Order that kept each step verifiable

1. **Look first.** `dotnet --list-sdks` and `dotnet --list-runtimes`. A test run needs the runtime of the framework the tests target: with only 10 and 11 installed, the old net9.0 tests abort with "You must install or update .NET" (`testhost.exe`), so there is no baseline to compare with. Say so in the report instead of pretending one exists.
2. **Framework and the packages that share its major.** `sed -i` the `<TargetFramework>` in every csproj; move `Microsoft.EntityFrameworkCore*`, `Microsoft.AspNetCore.Authentication.JwtBearer` to the same major (`11.0.0`), and `Microsoft.Extensions.Http.Resilience` and `Microsoft.Extensions.ServiceDiscovery` to the newest of that major (`11.0.1`). Build and test before touching anything else. Aspire 9.5.1 still ran on net11 at this point, which separates framework problems from Aspire problems.
3. **Then the third-party packages**, one family at a time, testing after each: Aspire, Swashbuckle and OpenApi, OpenTelemetry, MSTest.

Look versions up with `C:\Github\tools\nuget_versions.py` (`ID`, `--major N`, `--deps VERSION`), not with `curl` and `grep`. `--deps` shows the target frameworks: Aspire 17 libraries target net10.0, which a net11.0 project consumes without trouble.

## Aspire 9.x to 17

- The AppHost line `<Sdk Name="Aspire.AppHost.Sdk" Version="..."/>` still builds but warns `ASPIRE012` (the SDK is obsolete). Follow the warning: plain `Microsoft.NET.Sdk`, drop `<IsAspireHost>`, add `PackageReference` for `Aspire.Hosting.AppHost` and `Aspire.Hosting.Dotnet`, and remove the AppHost's `ProjectReference` to each service project.
- `AddProject<Projects.X>("name")` is obsolete (`CS0618`): `AddDotnetProject("name", "../X/X.csproj")`. The test project keeps its own references, and `Projects.<AppHost>` is still generated for `DistributedApplicationTestingBuilder`.
- `Aspire.Hosting.NodeJs` stops at 9.5.x. Use `Aspire.Hosting.JavaScript`.
- **`AddNpmApp` ran the `start` script; `AddJavaScriptApp` runs `dev`.** A UI without a `dev` script fails with `npm error Missing script: "dev"`, followed by a confusing `k8s ... resource is being deleted` log error from the teardown. Pass the script: `AddJavaScriptApp("angular", "../../UI", "start")` (the third argument is the run script name).
- The old 9.5.1 packages carried NU1903 and NU1904 vulnerability warnings (Aspire.Hosting, the dashboard SDK, MessagePack). They are gone at 17. Confirm with `dotnet list <sln> package --vulnerable --include-transitive`.
- `Aspire.Microsoft.EntityFrameworkCore.SqlServer`, `Aspire.Hosting.SqlServer` and `Aspire.Hosting.Testing` all move to the same 17.0.0. A transitive `Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore` on an older major was harmless here.

## Swashbuckle 9 to 11 and Microsoft.OpenApi 1.x to 3.x

- Swashbuckle 11 pulls `Microsoft.OpenApi` 3.x. `using Microsoft.OpenApi.Models;` becomes `using Microsoft.OpenApi;`.
- `AddSecurityDefinition("Bearer", new OpenApiSecurityScheme { Type, Scheme, BearerFormat, Description })` is unchanged. The requirement changes from `new OpenApiSecurityScheme { Reference = new OpenApiReference { Type = ReferenceType.SecurityScheme, Id = "Bearer" } }` to
  `swagger.AddSecurityRequirement(document => new OpenApiSecurityRequirement { [new OpenApiSecuritySchemeReference("Bearer", document)] = [] });`
- A wrong scheme name or a missing requirement still compiles, so add a test: start the API (Aspire testing host works), GET `/swagger/v1/swagger.json`, assert `components.securitySchemes.Bearer` has `type`, `scheme`, `bearerFormat`, and that `security[0]` names `Bearer`. Swagger is mapped in Development only, and the Aspire host runs Development.

## MSTest 3 to 4 with the MSTest runner

- With `EnableMSTestRunner` and OutputType Exe, plain `dotnet test` on SDK 10 or later stops with "Testing with VSTest target is no longer supported by Microsoft.Testing.Platform". Add a `global.json` with no sdk pin:
  `{ "test": { "runner": "Microsoft.Testing.Platform" } }`
  and run `dotnet test --solution X.sln` (the old `dotnet test X.sln` form fails in that mode). Mention it in the README; Visual Studio's Test Explorer is not affected.
- The existing tests needed no source change. Expect a few analyzer warnings at most.

## OpenTelemetry and the rest

- The OpenTelemetry family moves together (`1.21.0`: exporter, hosting, AspNetCore, Http, Runtime). `OpenTelemetry.Instrumentation.GrpcNetClient` has only beta releases: remove it when nothing uses gRPC (`grep` the sources), do not chase a beta.
- `dotnet list <sln> package --outdated` is the checklist for what is left; a package that says "Not found at the sources" is a prerelease-only package.

## Traps

- A build error right after another run usually is a file lock from a running AppHost or API. Stop it (`C:\Github\tools\stop_dev_server.py --port N`) and rebuild before suspecting the change.
- `sed -i 's#net9.0#net11.0#'` on the csproj files keeps the BOM and line endings. Check `git diff -w --ignore-cr-at-eol --stat`, not plain `git diff` (the CRLF warnings are noise).
- Update the README in the same pass: the .NET SDK line and download link, the Visual Studio line (a Visual Studio that supports the new .NET), the Aspire and Swagger lines. Keep a statement in the README only if it is true for the new versions.
- Finish by running the stack once through the AppHost project (`dotnet run --project <AppHost> --launch-profile https`): the unit tests do not start the front end.
