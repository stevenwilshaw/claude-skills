---
name: scaffold-fullstack-app
description: Scaffold a new full-stack TypeScript monorepo - Node/Express + Postgres (raw SQL, node-pg-migrate, no ORM) API and a Vite + React + Tailwind + TanStack (Router/Query/Form) SPA, with Docker, tests and a sample "Posts" feature wired end to end. Optionally adds BetterAuth or Keycloak (BFF) auth, MinIO image storage with thumbnails, and Storybook. Use this whenever the user wants to start, bootstrap, scaffold, initialise or "set up a new project/app/starter/boilerplate" with TypeScript + Node + React, a React SPA with an Express backend, or a monorepo/workspaces setup with a frontend and API - even if they don't name every technology.
---

# Scaffold a full-stack TypeScript app

Creates a complete, runnable npm-workspaces monorepo. The architecture is opinionated and deliberately plain: easy to read, no ORM, no DI framework, no UI framework. The reference files hold the exact conventions and code templates - follow them rather than inventing structure, because the user specified these conventions precisely.

## Stack at a glance

- **Workspaces**: `apps/api`, `apps/web`, `packages/shared` (zod schemas + DTO types used by both sides). Scope is `@app/*`.
- **API**: Express 5, zod validation in routes, Postgres via `pg` with raw SQL, `node-pg-migrate` (SQL migrations), Vitest (unit + integration with Testcontainers).
- **Web**: Vite, React, Tailwind, TanStack Router/Query/Form, axios, floating-ui, motion, lucide-react, react-select, Vitest + Testing Library. No UI framework.
- **Ops**: `docker-compose.yml` for local services, `.env.template`, single-container `Dockerfile` (Express serves the built SPA; API under `/api`; migrations run before start).

## Workflow

### 1. Ask the questions

Use AskUserQuestion with these in ONE call (they are independent):

1. **Auth** - "Which authentication do you need?" Options: `Local Users with BetterAuth`, `Keycloak IDP`. (The user can answer Other, e.g. "none" - then skip auth entirely.)
2. **Image storage** - "Do you need image storage?" Yes (MinIO) / No.
3. **Storybook** - "Do you need Storybook for the shared components?" Yes / No.
4. **Git** - "Initialise a git repository?" Options: `Initialise and make a first commit` (recommended), `Initialise only`, `No`. Skip this question (and do nothing git-related) if the target is already inside a git repo (`git rev-parse --is-inside-work-tree` succeeds). If the folder-location question is also needed (non-empty target), that makes four questions, which is the tool's limit - drop nothing, just keep both.

Project name: use the target directory's name unless the user gave one. Target is the current working directory unless told otherwise. Look at what is already there first; if it is not empty, ask before writing anything that would overwrite existing files.

### 2. Preflight

Run `node -v`, `npm -v`, `docker --version`. Node must be >= 22. Docker is needed for integration tests and compose; if it is missing, still scaffold but say that integration tests and `docker build` could not be verified.

### 3. Scaffold

Read the reference files and create files in this order. Read a reference right before you use it rather than all up front.

| Step | Read | Always? |
|---|---|---|
| Root tooling, compose, env, Dockerfile, shared package | `references/01-monorepo-and-tooling.md` | yes |
| API | `references/02-backend.md` | yes |
| Web | `references/03-frontend.md` | yes |
| Auth | `references/04-auth-betterauth.md` **or** `references/05-auth-keycloak.md` | only if chosen |
| Images | `references/06-image-storage.md` | only if chosen |
| Storybook | `references/07-storybook.md` | only if chosen |
| `CLAUDE.md` (written last, in step 6) | `references/08-claude-md.md` | yes |

Auth and images change the sample feature (author column, image columns, extra routes, extra form field), so apply them as modifications on top of the base Posts feature, following the "Changes to Posts" section in each reference. Write the migrations in dependency order (auth tables, then posts, then the image columns folded into the posts migration - do not create a separate migration for a brand-new table).

### 4. Install and use current versions

Do not pin versions from memory. Run `npm install <pkg>` / `npm install -D <pkg>` per workspace (e.g. `npm i -w apps/api express pg zod`) so the latest compatible versions are resolved, and the lockfile is created. The reference code was last verified against: TypeScript 7, Vite 8, Vitest 4, Express 5, Zod 4, Tailwind 4, React 19, tsup 8, node-pg-migrate 9, better-auth 1.x, TanStack Router file-based routing with the Vite plugin. If an installed major differs and an API has drifted, adapt to the installed version's docs rather than downgrading. Note the generated `routeTree.gen.ts` must be produced (`npx vite build` in `apps/web`) before the first typecheck.

### 5. Verify - do not skip

A scaffold that does not run is worse than none. Run, and fix until green:

1. `npm run typecheck`
2. `npm test` (unit tests, API + web)
3. `npm run test:int` (needs Docker running; Testcontainers starts Postgres, and MinIO if images were chosen)
4. `npm run build`
5. `npm run db:up` (compose; with MinIO this is a two-step command, see 06) then `npm run migrate`, start the API and `curl localhost:3000/api/health` and `/api/posts` (with auth, expect 401 for posts).
6. If Docker is available, `docker build -t <name> .` and make sure the container starts, migrates, and serves `/` and `/api/health`.

Smoke-test hints: the container needs every required env var of the chosen options (derive them from `config.ts`: DATABASE_URL, plus APP_URL/BETTER_AUTH_SECRET or OIDC_*/SESSION_SECRET, plus S3_ENDPOINT=http://minio:9000/S3_BUCKET/S3_ACCESS_KEY/S3_SECRET_KEY). BetterAuth sign-up via curl: `POST /api/auth/sign-up/email` with JSON `{name,email,password}` and header `Origin: $APP_URL` (it must be in `trustedOrigins`); keep the returned cookie (`-c jar -b jar`). Image upload: `curl -b jar -X PUT -F image=@file.png localhost:3000/api/posts/<id>/image`. On Windows/Git Bash, `/tmp` is not visible to Windows `node`, and MSYS rewrites leading `/` in `docker run`/`-w`/format args (set `MSYS_NO_PATHCONV=1`). For binary bodies in supertest (thumbnail), use `.buffer().parse((res, cb) => {...})` with `res` typed as supertest's `Response`.

Report honestly what passed and what could not be run (e.g. Docker not running). Never claim the app works if a step was skipped.

### 6. Wrap up

Write a short root `README.md` (prerequisites, `cp .env.template .env`, `npm install`, `npm run db:up`, `npm run migrate`, `npm run dev`, test commands, docker build/run, how to add a feature, chosen options and any manual steps such as the Keycloak dev login). Also write a `CLAUDE.md` at the project root from `references/08-claude-md.md` (after verification, so the commands in it are known to work). It records the architecture rules and gotchas so that future Claude sessions in the project keep to them instead of introducing an ORM or putting logic in handlers; trim it to the options chosen. Then, as the very last file operation, handle git according to the answer to question 4 (a commit records history under the user's identity, so only do it when they said yes):
- `git init -b main` (only when the target is not already in a repo).
- If a first commit was chosen: run `git status` and check that `.env`, `node_modules`, `dist`, `coverage` and `storybook-static` are ignored and `package-lock.json` and `.env.template` are included. Check `git config user.name` / `user.email` are set; if not, do not invent an identity - leave the files staged and tell the user. Otherwise `git add -A` and commit with the message `Initial scaffold` (plus whatever attribution trailer the session instructs for commits).
- Never add a remote and never push.

Then tell the user the final folder layout, the commands, and any decisions you made (e.g. Storybook lives inside `apps/web`, see its reference for why).

## Conventions that matter (and why)

- **Colocation**: put a file next to its only user; move it up to the nearest common ancestor only when a second consumer appears. This keeps features deletable as a unit.
- **API feature = fixed set of files**: `*.routes.ts` (function, zod validation), `*.handlers.ts` (function, maps HTTP to controller calls), `*.controller.ts` (class, constructor-injected repository/services, no `req`/`res`), `*.repository.ts` (class, `Db` injected, raw SQL). The seam between handlers and controller is what keeps business logic testable without HTTP.
- **Wiring** happens by hand in one `container.ts`. No DI framework, no decorators.
- **Repositories** return plain objects using SQL aliases (`created_at AS "createdAt"`). No mapper layer, no query builder, no base repository class.
- **Frontend routes are thin**: files in `src/routes` only declare the route (loader, params, component). All UI lives in `src/features/<feature>/<list|view|create|edit>/`.
- **No forms on list pages**: lists only list. Add/edit forms go on their own screen (route) or in a modal, never inline on the list page. Applies to every feature you add, not just Posts.
- **Inputs vs fields**: `components/inputs/TextInput` knows nothing about forms (plain props). `components/form/TextField` wraps it with TanStack Form's `useFieldContext` and adds label + error. Same pattern for every input type.
- Keep generated code short and boring. Comments only where the reason is non-obvious.
