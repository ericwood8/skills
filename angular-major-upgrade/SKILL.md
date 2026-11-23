---
name: angular-major-upgrade
description: Use when upgrading an Angular app several majors (for example 18 to 22) with ng update, or when a build fails after an upgrade because a library uses a removed API, or when moving from @angular-devkit/build-angular to @angular/build. Trigger words - ng update, No matching export ComponentFactoryResolver, ngx-toastr peer dependency, ERESOLVE, overrides, @angular/build, initial bundle budget, ChangeDetectionStrategy.Eager, provideZoneChangeDetection, standalone true removed, npx ng.
---

# Upgrading an Angular app several majors, one at a time

Worked through on an Angular 18 Material app (standalone components, karma tests, ngx-toastr) taken to 22.3: 42 specs passed before and after every step.

## Before the first step

- The working tree must be clean (`git status`): `ng update` refuses otherwise, and a clean tree is the way back (`git checkout .` and `npm ci`).
- Record the baseline: `python C:/Github/tools/run_frontend_tests.py <app folder>` (the test count) and `npx ng build` (the bundle size and warnings). `npm view @angular/core dist-tags` shows the newest major.

## One major per step

```
npx ng update @angular/core@19 @angular/cli@19 @angular/material@19 --force
npx ng build
python C:/Github/tools/run_frontend_tests.py <app folder>
```

Repeat for 20, 21, 22. From the second step the tree is dirty, so add `--allow-dirty`. Check that each step ends with the same test count before starting the next, and read the list of files a migration changed (`git diff -w --ignore-cr-at-eol`). Migrations seen on 18 to 22, all behaviour preserving:

- 19: removes `standalone: true` (the default now).
- 20: edits `angular.json` and `tsconfig.json`.
- 21: `*ngFor` and `*ngIf` become `@for (x of xs; track x)` and `@if`; `CommonModule` leaves the imports; `main.ts` gets an explicit `provideZoneChangeDetection()` because new apps are zoneless.
- 22: every component gets `changeDetection: ChangeDetectionStrategy.Eager` (OnPush is the new default); spec files and the tsconfigs are touched; TypeScript moves to 6.

A step takes a minute or two: run it in a plain Bash call, or wrap it with `python C:/Github/tools/run_logged.py <log> -- npx ng update ...` and `run_in_background: true`.

## A library that breaks on the new major

- Symptom: `No matching export in ".../@angular/core/fesm2022/core.mjs" for import "ComponentFactoryResolver"` (an API that Angular 22 removed). Find the importer: `grep -rl ComponentFactoryResolver node_modules --include=*.mjs` outside `@angular/*`. Here it was `ngx-toastr` 19.
- Check the fix before installing: `npm pack <pkg>@<version>` into a scratch folder, extract, and `grep` its `fesm2022` file for the removed name (ngx-toastr 20.0.5 only mentions it in a comment).
- If the new release declares the previous Angular major as its peer (ngx-toastr 20: `^21`), `npm install` stops with ERESOLVE. Do not use `--legacy-peer-deps` (it hides every later conflict). Add an `overrides` entry to `package.json` so the peer follows your versions:
  `"overrides": { "ngx-toastr": { "@angular/core": "$@angular/core", "@angular/common": "$@angular/common" } }`
  and note in the README why it is there and when to remove it.

## Moving to @angular/build

When the project still uses `@angular-devkit/build-angular`, the dev server warns that the Webpack builder is deprecated. In `angular.json` the builder names change one for one: `:application`, `:dev-server`, `:extract-i18n`, `:karma` under `@angular/build:` (`sed 's#@angular-devkit/build-angular:#@angular/build:#'`). Then `npm uninstall @angular-devkit/build-angular` and `npm install -D @angular/build@^22.x`. Options in `angular.json` stay valid; build and tests pass unchanged.

## Bundle budget

The production budget (`budgets`, type `initial`) warns past the limit and fails past the error limit. A major upgrade grows the bundle slightly (771 kB after 19, 818 kB after 22 for this app). Raise the numbers to fit the app (`1mb` warning, `2mb` error) in the same change instead of leaving a warning on every build.

## Prove it runs, not just builds

Unit specs do not show a broken app. Run the dev server against a real API (see `verify-in-running-app`): `proxy.conf.js` here reads `services__timeentryapi__http__0`, so start `ng serve --port N` with that variable set to the API's URL and `NODE_ENV=development`. Load the app, sign in, open every screen, add, edit and delete one row, and read the console and the network list for errors. Stop the dev server by its port afterwards.

## Finish

- `npm audit --omit=dev` (production dependencies; dev tooling findings are separate and usually come with the CLI).
- The README states the Angular version and any `overrides` entry; the repository's other docs name the same version.
- Report what was not clicked through (screens that write data, screens without a menu entry).
