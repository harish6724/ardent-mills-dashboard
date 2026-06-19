# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Stack

- **Framework**: Angular 2+ (TypeScript)
- **Package manager**: npm
- **Build tool**: Angular CLI (`@angular/cli`)
- **Test runner**: Karma + Jasmine (unit), e2e via `ng e2e`
- **Linter**: ESLint (`@angular-eslint`)

## Setup

### Prerequisites
- Node.js (active LTS or maintenance LTS) — includes npm
- **Windows PowerShell only**: if scripts are disabled, run once:
  ```powershell
  Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
  ```

### Install Angular CLI globally
```bash
npm install -g @angular/cli
```

### Create a new workspace
```bash
ng new my-app
cd my-app
```

## Common Commands

```bash
# Start dev server with auto-reload (http://localhost:4200)
ng serve
ng serve --open          # also opens the browser

# Build
ng build                                    # development build
ng build --configuration production         # production build

# Test
ng test                                     # run unit tests (watch mode)
ng e2e                                      # build, serve, then run e2e tests

# Lint
ng lint

# Check CLI version
ng version
```

## Scaffolding with Angular CLI

```bash
# Aliases: ng g = ng generate
ng generate component features/my-feature
ng generate service core/services/my-service
ng generate module features/my-feature --routing
ng generate pipe shared/pipes/my-pipe
ng generate guard core/guards/my-guard
```

## CLI Option Conventions (from angular.dev/tools/cli)

- Boolean flags: `--flag` (true) / `--no-flag` (false)
- Array options: `--option val1 val2` or `--option val1 --option val2`
- File paths: absolute or relative to workspace/project root

## Dependency & Configuration Management

```bash
ng add <package>        # add and configure an external library
ng update               # update workspace and Angular dependencies
ng config <key> <value> # read/write values in angular.json
ng run <target>         # run a custom Architect/builder target
```

## Architecture

Angular 2+ uses a module/component tree. Recommended structure:

```
src/
  app/
    core/             # Singleton services, guards, interceptors (imported once in AppModule)
    shared/           # Shared components, pipes, directives (re-exported via SharedModule)
    features/         # Feature modules — one folder per route/domain area
      feature-a/
        components/
        services/
        feature-a.module.ts
        feature-a-routing.module.ts
    app-routing.module.ts
    app.module.ts
  environments/       # environment.ts / environment.prod.ts
```

Key conventions:
- **Modules**: `CoreModule` for app-wide singletons, `SharedModule` for reusable UI, feature modules for lazy-loaded routes.
- **Lazy loading**: feature modules should be loaded via `loadChildren` in `app-routing.module.ts` to keep the initial bundle small.
- **Services**: provided in `root` (or `CoreModule`) unless scoped to a feature. Use `HttpClient` (never raw `fetch`) inside services, not components.
- **Reactive patterns**: prefer `Observable` + `async` pipe in templates over manual `.subscribe()`. Unsubscribe with `takeUntilDestroyed()` (Angular 16+) or `DestroyRef`.
- **Change detection**: use `OnPush` for components that receive data only via `@Input()` to improve performance.
- **Signals** (Angular 17+): prefer `signal()` / `computed()` / `effect()` for local reactive state over `BehaviorSubject`.
- **Standalone components** (Angular 14+): new components may omit `NgModule` by using `standalone: true` and importing dependencies directly.
