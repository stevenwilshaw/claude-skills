# Monorepo, tooling, Docker

## Layout

```
.
├── apps/
│   ├── api/            # Express API (see 02-backend.md)
│   └── web/            # Vite SPA   (see 03-frontend.md)
├── packages/
│   └── shared/         # zod schemas + DTO types shared by api and web
├── docker/             # service config (e.g. keycloak realm)
├── docker-compose.yml  # local dev services (postgres, + minio / keycloak if chosen)
├── Dockerfile
├── .dockerignore
├── .env.template
├── .gitignore
├── package.json
├── tsconfig.base.json
└── README.md
```

## Root `package.json`

```json
{
  "name": "<project-name>",
  "private": true,
  "type": "module",
  "workspaces": ["apps/*", "packages/*"],
  "engines": { "node": ">=22" },
  "scripts": {
    "dev": "concurrently -n api,web -c blue,green \"npm run dev -w apps/api\" \"npm run dev -w apps/web\"",
    "db:up": "docker compose up -d --wait",
    "db:down": "docker compose down",
    "migrate": "npm run migrate -w apps/api",
    "migrate:create": "npm run migrate:create -w apps/api --",
    "build": "npm run build -w apps/web && npm run build -w apps/api",
    "typecheck": "npm run typecheck --workspaces --if-present",
    "test": "npm run test --workspaces --if-present",
    "test:int": "npm run test:int -w apps/api"
  }
}
```

`npm i -D concurrently typescript` at the root.

## `tsconfig.base.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

Each workspace has its own `tsconfig.json` that `extends` it.

## `packages/shared`

`package.json`: `{ "name": "@app/shared", "private": true, "type": "module", "exports": { ".": "./src/index.ts" }, "scripts": { "typecheck": "tsc --noEmit" } }`, dependency `zod`.

It exports TypeScript source directly. Vite and Vitest compile it as-is; the API build (tsup) bundles it in with `--noExternal @app/shared`, so nothing has to be published or built separately. Both apps list `"@app/shared": "*"` in dependencies.

Add `src/index.ts` re-exporting feature schemas, e.g. `export * from './posts';` (see 02-backend.md for `posts.ts`).

## `.env.template`

Defaults that work out of the box once copied to `.env`. Compose, the API (`--env-file`), `node-pg-migrate` (via dotenv-cli) and Vite (`envDir`) all read this one root file.

```dotenv
# --- API ---
NODE_ENV=development
PORT=3000
DATABASE_URL=postgres://app:app@localhost:5432/app

# --- Web (Vite dev server; proxies /api to the API) ---
WEB_PORT=5173

# --- Postgres container (docker-compose) ---
POSTGRES_USER=app
POSTGRES_PASSWORD=app
POSTGRES_DB=app
POSTGRES_PORT=5432
```

Keep `DATABASE_URL`'s port in sync with `POSTGRES_PORT` (and, with auth, `APP_URL`'s port with `WEB_PORT`) - add a comment saying so at the top of the file, since they are separate variables.

Auth / MinIO references append their own variables here. Keep `.env` in `.gitignore`, commit `.env.template`.

## `docker-compose.yml` (local development)

```yaml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-app}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-app}
      POSTGRES_DB: ${POSTGRES_DB:-app}
    ports:
      - "${POSTGRES_PORT:-5432}:5432"
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 3s
      timeout: 3s
      retries: 20

volumes:
  db-data:
```

The 05 / 06 references add `keycloak`, `minio`, `minio-init` services and volumes. This file is only for local dev; integration tests use Testcontainers and never touch it.

## `Dockerfile`

One image, one process: Express serves the built SPA from `/app/public` and the API on `/api`. No nginx. Migrations are copied in and run before the server starts.

```dockerfile
# ---- build ----
FROM node:22-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
COPY apps/api/package.json apps/api/
COPY apps/web/package.json apps/web/
COPY packages/shared/package.json packages/shared/
RUN npm ci
COPY . .
RUN npm run build

# ---- runtime ----
FROM node:22-slim AS runtime
ENV NODE_ENV=production PORT=3000 STATIC_DIR=/app/public
WORKDIR /app
COPY package.json package-lock.json ./
COPY apps/api/package.json apps/api/
COPY apps/web/package.json apps/web/
COPY packages/shared/package.json packages/shared/
RUN npm ci --omit=dev --workspace apps/api
COPY --from=build /app/apps/api/dist ./apps/api/dist
COPY --from=build /app/apps/web/dist ./public
COPY apps/api/migrations ./apps/api/migrations
COPY docker-entrypoint.sh ./
RUN chmod +x docker-entrypoint.sh
# Keep the workspace layout: npm may nest a dependency under apps/api/node_modules, and Node only
# finds it if the bundle runs from inside apps/api.
WORKDIR /app/apps/api
USER node
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD node -e "fetch('http://localhost:'+process.env.PORT+'/api/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
ENTRYPOINT ["/app/docker-entrypoint.sh"]
```

`docker-entrypoint.sh` (LF line endings - add `*.sh text eol=lf` to `.gitattributes`):

```sh
#!/bin/sh
set -e
npx --no-install node-pg-migrate up --migrations-dir migrations
exec node dist/index.js
```

`npm ci --omit=dev --workspace apps/api` may still install some root-level dev tooling; that is acceptable, but if image size matters, prune afterwards with `npm prune --omit=dev`.

### Running the image locally (verify step)

The container needs the same required env vars as the API and a route to the database. With compose project `<name>` (use `docker compose -p <name>`, so parallel projects do not clash) the db is reachable on the compose network:
```bash
docker build -t <name> .
docker run --rm -p 3000:3000 --network <name>_default \
  -e DATABASE_URL=postgres://app:app@db:5432/app \
  -e APP_URL=http://localhost:3000 -e BETTER_AUTH_SECRET=<32+ chars> \
  <name>
```
(Use the env vars of the chosen auth/image options; without a required var `loadConfig` throws at startup.) Then check `/`, `/posts` (SPA fallback) and `/api/health`.

`node-pg-migrate` and `pg` must be in the API's `dependencies` (not dev) so they survive `--omit=dev`. `slim` (glibc) rather than alpine avoids native-module trouble (sharp).

`.dockerignore`: `node_modules`, `**/node_modules`, `**/dist`, `.env`, `.git`, `*.log`, `**/coverage`, `**/storybook-static`.

## `.gitignore`

`node_modules`, `dist`, `.env`, `coverage`, `storybook-static`, `*.log`, `.DS_Store`.
