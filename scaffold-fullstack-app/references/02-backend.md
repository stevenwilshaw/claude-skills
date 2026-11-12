# Backend (`apps/api`)

Express 5, ESM, TypeScript. Dev runner `tsx`, production build `tsup`.

## Packages

- deps: `express pg zod node-pg-migrate @app/shared`
- dev: `typescript tsx tsup vitest supertest @types/express @types/pg @types/supertest @types/node testcontainers @testcontainers/postgresql dotenv-cli`

## `package.json` scripts

```json
{
  "name": "@app/api",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "tsx watch --env-file=../../.env src/index.ts",
    "build": "tsup",
    "start": "node dist/index.js",
    "migrate": "dotenv -e ../../.env -- node-pg-migrate up --migrations-dir migrations",
    "migrate:down": "dotenv -e ../../.env -- node-pg-migrate down --migrations-dir migrations",
    "migrate:create": "node-pg-migrate create --migrations-dir migrations --migration-file-language sql",
    "typecheck": "tsc --noEmit",
    "test": "vitest run --project unit",
    "test:int": "vitest run --project integration"
  }
}
```

Migrations are plain `.sql` files with `-- Up Migration` / `-- Down Migration` markers. SQL needs no compile step, so the Docker image can run them directly, and it fits the "no abstractions" goal. `node-pg-migrate` reads `DATABASE_URL` from the environment.

`tsup.config.ts` (tsup has no CLI flag for `noExternal`, so it must be a config file):
```ts
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/index.ts'],
  format: ['esm'],
  target: 'node22',
  clean: true,
  noExternal: ['@app/shared'], // shared ships TS source, so bundle it in
});
```

`tsconfig.json`: extends the base, `"types": ["node"]`, `include: ["src", "test", "vitest.config.ts", "tsup.config.ts"]`, `noEmit: true`.

Create files with the Write tool rather than big shell heredocs - heredocs containing `$${...}`/`$(...)` can be blocked by shell safety checks.

Migration timestamps: `migrate:create` prefixes the current time in ms, so two created back to back sort correctly, but if you write migration files by hand, give them ascending timestamps in dependency order (auth tables before posts).

## Layout

```
apps/api/
├── migrations/
│   └── <timestamp>_create-posts.sql
├── src/
│   ├── index.ts                  # entry: config, pool, container, listen, shutdown
│   ├── app.ts                    # createApp(): middleware + route mounting + static SPA
│   ├── config.ts                 # zod-parsed env
│   ├── container.ts              # manual wiring of repositories, services, controllers
│   ├── db/pool.ts                # createPool, Db type
│   ├── http/
│   │   ├── errors.ts             # HttpError
│   │   ├── error-handler.ts
│   │   └── validate.ts           # zod validation middleware
│   └── features/
│       └── posts/
│           ├── posts.routes.ts
│           ├── posts.handlers.ts
│           ├── posts.controller.ts
│           ├── posts.repository.ts
│           ├── posts.controller.test.ts       # unit (fake repository)
│           ├── posts.repository.int.test.ts   # integration (real db)
│           └── posts.routes.int.test.ts       # integration (full app, supertest)
├── test/
│   ├── global-setup.ts           # starts Postgres container, runs migrations
│   └── db.ts                     # test pool + resetDb
└── vitest.config.ts
```

A new feature = a new folder under `features/` with the same four files (+ tests), one line in `container.ts`, one line in `app.ts`.

## Shared schema - `packages/shared/src/posts.ts`

```ts
import { z } from 'zod';

export const postStatusSchema = z.enum(['draft', 'published']);
export type PostStatus = z.infer<typeof postStatusSchema>;

export const createPostSchema = z.object({
  title: z.string().trim().min(1, 'Title is required').max(200),
  description: z.string().trim().max(5000).default(''),
  status: postStatusSchema.default('draft'),
});
export const updatePostSchema = createPostSchema;
export type CreatePostInput = z.input<typeof createPostSchema>;

export interface Post {
  id: string;
  title: string;
  description: string;
  status: PostStatus;
  createdAt: string; // ISO string - see createPool
  updatedAt: string;
}
```

(The web form validates with the same schema. If `default()` makes the form's input/output types awkward, drop the defaults and let the form supply them via `defaultValues`.)

## `src/config.ts`

```ts
import { z } from 'zod';

const schema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().min(1),
  STATIC_DIR: z.string().optional(), // set in Docker; the built SPA to serve
});

export type Config = z.infer<typeof schema>;
export const loadConfig = (env: NodeJS.ProcessEnv = process.env): Config => schema.parse(env);
```

Auth / images references extend this schema.

## `src/db/pool.ts`

```ts
import pg from 'pg';

export type Db = pg.Pool;

const TIMESTAMPTZ = 1184;

/**
 * By default timestamptz columns come back as Date objects, which JSON-serialise to ISO
 * strings - so our DTO types would lie. Parsing to ISO strings up front keeps the types honest.
 * Libraries that expect Date objects (better-auth) get a pool with `dates: 'date'`.
 */
export function createPool(connectionString: string, opts: { dates?: 'iso' | 'date' } = {}): Db {
  if (opts.dates === 'date') return new pg.Pool({ connectionString });
  return new pg.Pool({
    connectionString,
    types: {
      getTypeParser: ((oid: number, format?: 'text' | 'binary') =>
        oid === TIMESTAMPTZ
          ? (v: string) => new Date(v).toISOString()
          : pg.types.getTypeParser(oid, format as 'text')) as typeof pg.types.getTypeParser,
    },
  });
}
```

## HTTP helpers

`src/http/errors.ts`
```ts
export class HttpError extends Error {
  constructor(public readonly status: number, message: string) {
    super(message);
  }
}
```

`src/http/validate.ts` - Express 5 makes `req.query` read-only, so validated input goes to `res.locals.valid` instead of being written back.
```ts
import type { RequestHandler, Response } from 'express';
import type { ZodType } from 'zod';

interface Schemas {
  params?: ZodType;
  query?: ZodType;
  body?: ZodType;
}

export const validate =
  (schemas: Schemas): RequestHandler =>
  (req, res, next) => {
    // A thrown ZodError is turned into a 400 by the error handler.
    res.locals.valid = {
      params: schemas.params?.parse(req.params),
      query: schemas.query?.parse(req.query),
      body: schemas.body?.parse(req.body),
    };
    next();
  };

/** Typed accessor for what `validate` stored. */
export const valid = <T extends { params?: unknown; query?: unknown; body?: unknown }>(res: Response) =>
  res.locals.valid as T;
```

`src/http/error-handler.ts`
```ts
import type { ErrorRequestHandler } from 'express';
import { ZodError } from 'zod';
import { HttpError } from './errors';

export const errorHandler: ErrorRequestHandler = (err, _req, res, _next) => {
  if (err instanceof ZodError) {
    res.status(400).json({ error: 'Validation failed', issues: err.issues });
    return;
  }
  if (err instanceof HttpError) {
    res.status(err.status).json({ error: err.message });
    return;
  }
  console.error(err);
  res.status(500).json({ error: 'Internal server error' });
};
```

## The Posts feature

### `posts.repository.ts`

Raw SQL, aliases give camelCase, no mapper. One place to read to understand every query.

```ts
import type { CreatePostInput, Post } from '@app/shared';
import type { Db } from '../../db/pool';

const COLUMNS = `id, title, description, status, created_at AS "createdAt", updated_at AS "updatedAt"`;

export class PostsRepository {
  constructor(private readonly db: Db) {}

  async list(): Promise<Post[]> {
    const { rows } = await this.db.query<Post>(`SELECT ${COLUMNS} FROM posts ORDER BY created_at DESC`);
    return rows;
  }

  async findById(id: string): Promise<Post | null> {
    const { rows } = await this.db.query<Post>(`SELECT ${COLUMNS} FROM posts WHERE id = $1`, [id]);
    return rows[0] ?? null;
  }

  async create(input: Required<CreatePostInput>): Promise<Post> {
    const { rows } = await this.db.query<Post>(
      `INSERT INTO posts (title, description, status) VALUES ($1, $2, $3) RETURNING ${COLUMNS}`,
      [input.title, input.description, input.status],
    );
    return rows[0]!;
  }

  async update(id: string, input: Required<CreatePostInput>): Promise<Post | null> {
    const { rows } = await this.db.query<Post>(
      `UPDATE posts SET title = $2, description = $3, status = $4, updated_at = now()
       WHERE id = $1 RETURNING ${COLUMNS}`,
      [id, input.title, input.description, input.status],
    );
    return rows[0] ?? null;
  }

  async delete(id: string): Promise<boolean> {
    const result = await this.db.query(`DELETE FROM posts WHERE id = $1`, [id]);
    return (result.rowCount ?? 0) > 0;
  }
}
```

### `posts.controller.ts`

Class, dependencies via constructor, no knowledge of Express. Add services to the constructor when logic outgrows the repository (`constructor(repo, imageService)`).

```ts
import type { Post } from '@app/shared';
import type { z } from 'zod';
import { createPostSchema } from '@app/shared';
import { HttpError } from '../../http/errors';
import type { PostsRepository } from './posts.repository';

type PostInput = z.output<typeof createPostSchema>;

export class PostsController {
  constructor(private readonly posts: PostsRepository) {}

  list(): Promise<Post[]> {
    return this.posts.list();
  }

  async get(id: string): Promise<Post> {
    return this.mustExist(await this.posts.findById(id));
  }

  create(input: PostInput): Promise<Post> {
    return this.posts.create(input);
  }

  async update(id: string, input: PostInput): Promise<Post> {
    return this.mustExist(await this.posts.update(id, input));
  }

  async remove(id: string): Promise<void> {
    if (!(await this.posts.delete(id))) throw new HttpError(404, 'Post not found');
  }

  private mustExist(post: Post | null): Post {
    if (!post) throw new HttpError(404, 'Post not found');
    return post;
  }
}
```

### `posts.handlers.ts`

A function that takes the controller and returns plain handlers. Handlers translate HTTP <-> controller and nothing else. Express 5 forwards rejected promises to the error handler, so no try/catch.

```ts
import type { CreatePostInput } from '@app/shared';
import type { RequestHandler } from 'express';
import { valid } from '../../http/validate';
import type { PostsController } from './posts.controller';

type Id = { params: { id: string } };
type WithBody = { body: Required<CreatePostInput> };

export function createPostsHandlers(controller: PostsController) {
  return {
    list: (async (_req, res) => {
      res.json(await controller.list());
    }) as RequestHandler,

    get: (async (_req, res) => {
      res.json(await controller.get(valid<Id>(res).params.id));
    }) as RequestHandler,

    create: (async (_req, res) => {
      res.status(201).json(await controller.create(valid<WithBody>(res).body));
    }) as RequestHandler,

    update: (async (_req, res) => {
      const { params, body } = valid<Id & WithBody>(res);
      res.json(await controller.update(params.id, body));
    }) as RequestHandler,

    remove: (async (_req, res) => {
      await controller.remove(valid<Id>(res).params.id);
      res.status(204).end();
    }) as RequestHandler,
  };
}

export type PostsHandlers = ReturnType<typeof createPostsHandlers>;
```

### `posts.routes.ts`

A function that takes handlers and returns a Router. This is where zod validation lives.

```ts
import { createPostSchema, updatePostSchema } from '@app/shared';
import { Router } from 'express';
import { z } from 'zod';
import { validate } from '../../http/validate';
import type { PostsHandlers } from './posts.handlers';

const idParams = z.object({ id: z.uuid() });

export function postsRoutes(h: PostsHandlers): Router {
  const router = Router();
  router.get('/', h.list);
  router.get('/:id', validate({ params: idParams }), h.get);
  router.post('/', validate({ body: createPostSchema }), h.create);
  router.put('/:id', validate({ params: idParams, body: updatePostSchema }), h.update);
  router.delete('/:id', validate({ params: idParams }), h.remove);
  return router;
}
```

### Migration `migrations/<timestamp>_create-posts.sql`

Create it with `npm run migrate:create -- create-posts` so the timestamp prefix is correct, then fill it in.

```sql
-- Up Migration
CREATE TABLE posts (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  title       text NOT NULL,
  description text NOT NULL DEFAULT '',
  status      text NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'published')),
  created_at  timestamptz NOT NULL DEFAULT now(),
  updated_at  timestamptz NOT NULL DEFAULT now()
);

-- Down Migration
DROP TABLE posts;
```

## Wiring

`src/container.ts` - the only place that knows how things are assembled.
```ts
import type { Config } from './config';
import type { Db } from './db/pool';
import { PostsController } from './features/posts/posts.controller';
import { PostsRepository } from './features/posts/posts.repository';

export function buildContainer(_config: Config, db: Db) {
  const postsRepository = new PostsRepository(db);
  const postsController = new PostsController(postsRepository);
  return { postsController };
}
export type Container = ReturnType<typeof buildContainer>;
```

`src/app.ts`
```ts
import { existsSync } from 'node:fs';
import path from 'node:path';
import express from 'express';
import type { Config } from './config';
import type { Container } from './container';
import { createPostsHandlers } from './features/posts/posts.handlers';
import { postsRoutes } from './features/posts/posts.routes';
import { errorHandler } from './http/error-handler';

export function createApp(config: Config, container: Container) {
  const app = express();
  app.disable('x-powered-by');
  app.use(express.json());

  const api = express.Router();
  api.get('/health', (_req, res) => void res.json({ status: 'ok' }));
  api.use('/posts', postsRoutes(createPostsHandlers(container.postsController)));
  api.use((_req, res) => void res.status(404).json({ error: 'Not found' }));
  app.use('/api', api);

  // Serve the built SPA (Docker). Anything that is not /api falls back to index.html for client routing.
  if (config.STATIC_DIR && existsSync(config.STATIC_DIR)) {
    const dir = path.resolve(config.STATIC_DIR);
    app.use(express.static(dir));
    app.get('/{*splat}', (_req, res) => void res.sendFile(path.join(dir, 'index.html')));
  }

  app.use(errorHandler);
  return app;
}
```

`src/index.ts`
```ts
import { createApp } from './app';
import { loadConfig } from './config';
import { buildContainer } from './container';
import { createPool } from './db/pool';

const config = loadConfig();
const pool = createPool(config.DATABASE_URL);
const app = createApp(config, buildContainer(config, pool));

const server = app.listen(config.PORT, () => console.log(`API listening on :${config.PORT}`));

const shutdown = () => server.close(() => void pool.end().then(() => process.exit(0)));
process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);
```

## Testing

`vitest.config.ts` - two projects so `npm test` stays fast and Docker-free:
```ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    projects: [
      {
        test: {
          name: 'unit',
          include: ['src/**/*.test.ts'],
          exclude: ['src/**/*.int.test.ts'],
        },
      },
      {
        test: {
          name: 'integration',
          include: ['src/**/*.int.test.ts'],
          globalSetup: ['./test/global-setup.ts'],
          fileParallelism: false, // tests share one database
          testTimeout: 30_000,
          hookTimeout: 120_000, // first run pulls the postgres image
        },
      },
    ],
  },
});
```

`test/global-setup.ts` - one real Postgres per run, migrated with the real migration files:
```ts
import path from 'node:path';
import { fileURLToPath } from 'node:url';
import { PostgreSqlContainer } from '@testcontainers/postgresql';
import { runner } from 'node-pg-migrate';
import type { TestProject } from 'vitest/node';

export default async function setup(project: TestProject) {
  const container = await new PostgreSqlContainer('postgres:17').start();
  const databaseUrl = container.getConnectionUri();

  await runner({
    databaseUrl,
    dir: path.resolve(path.dirname(fileURLToPath(import.meta.url)), '../migrations'),
    direction: 'up',
    migrationsTable: 'pgmigrations',
    log: () => {},
  });

  project.provide('databaseUrl', databaseUrl);
  return () => container.stop();
}

declare module 'vitest' {
  export interface ProvidedContext {
    databaseUrl: string;
  }
}
```

`test/db.ts`
```ts
import { inject } from 'vitest';
import { createPool, type Db } from '../src/db/pool';

export const createTestPool = () => createPool(inject('databaseUrl'));

/** Empty every table except the migration bookkeeping one, so tests stay independent. */
export async function resetDb(db: Db) {
  const { rows } = await db.query<{ t: string | null }>(
    `SELECT string_agg(format('%I', tablename), ', ') AS t
       FROM pg_tables WHERE schemaname = 'public' AND tablename <> 'pgmigrations'`,
  );
  if (rows[0]?.t) await db.query(`TRUNCATE ${rows[0].t} RESTART IDENTITY CASCADE`);
}
```

Tests to write for Posts:

- `posts.controller.test.ts` (unit): construct `new PostsController(fakeRepo)` where `fakeRepo` is an object literal with `vi.fn()` members cast `as unknown as PostsRepository`. Cover: `get` throws `HttpError` 404 when missing, `remove` 404 when nothing deleted, `create` passes input through.
- `posts.repository.int.test.ts`: real pool; `beforeEach(() => resetDb(db))`, `afterAll(() => db.end())`. Create, list order (newest first), findById, update bumps `updatedAt`, delete.
- `posts.routes.int.test.ts`: `createApp(config, buildContainer(config, db))` + `supertest`. Cover 201 create, 400 on empty title, 400 on non-uuid id, 404 unknown id, full CRUD round trip. Build `config` with `loadConfig({ DATABASE_URL: inject('databaseUrl'), NODE_ENV: 'test', ...dummyAuthOrS3Vars })` - use the real test database URL, not a placeholder: `buildContainer` may create real clients (auth pool, S3) that actually connect, and a fake URL turns expected 401s into 500s. Include every required env var of the chosen options (e.g. `APP_URL`, `BETTER_AUTH_SECRET`).

Integration tests need Docker running; say so in the README.
