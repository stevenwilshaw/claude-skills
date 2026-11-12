# Auth option A: Local users with BetterAuth

BetterAuth runs inside the Express API, stores users/sessions in the same Postgres database, and authenticates the SPA through its own httpOnly session cookie. Email + password sign-up/sign-in.

## Packages

- api: `better-auth`
- web: `better-auth`

## Env (append to `.env.template`, extend `config.ts` zod schema)

```dotenv
# --- Auth (BetterAuth) ---
# Browser-facing origin of the app. In dev the SPA runs on the Vite port and proxies /api.
APP_URL=http://localhost:5173
# Generate with: openssl rand -base64 32
BETTER_AUTH_SECRET=dev-only-change-me-dev-only-change-me
```
`config.ts`: add `APP_URL: z.string().url()`, `BETTER_AUTH_SECRET: z.string().min(32)`. Tests pass dummy values to `loadConfig`.

## Migration (create BEFORE the posts migration so the FK works)

`npm run migrate:create -- better-auth`. These are BetterAuth's core tables for Postgres with its default (camelCase, quoted) column names. Before finalising, cross-check against `npx @better-auth/cli generate` output for the installed version and adjust if it differs.

```sql
-- Up Migration
CREATE TABLE "user" (
  "id"            text PRIMARY KEY,
  "name"          text NOT NULL,
  "email"         text NOT NULL UNIQUE,
  "emailVerified" boolean NOT NULL DEFAULT false,
  "image"         text,
  "createdAt"     timestamptz NOT NULL DEFAULT now(),
  "updatedAt"     timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE "session" (
  "id"        text PRIMARY KEY,
  "expiresAt" timestamptz NOT NULL,
  "token"     text NOT NULL UNIQUE,
  "createdAt" timestamptz NOT NULL DEFAULT now(),
  "updatedAt" timestamptz NOT NULL DEFAULT now(),
  "ipAddress" text,
  "userAgent" text,
  "userId"    text NOT NULL REFERENCES "user" ("id") ON DELETE CASCADE
);

CREATE TABLE "account" (
  "id"                    text PRIMARY KEY,
  "accountId"             text NOT NULL,
  "providerId"            text NOT NULL,
  "userId"                text NOT NULL REFERENCES "user" ("id") ON DELETE CASCADE,
  "accessToken"           text,
  "refreshToken"          text,
  "idToken"               text,
  "accessTokenExpiresAt"  timestamptz,
  "refreshTokenExpiresAt" timestamptz,
  "scope"                 text,
  "password"              text,
  "createdAt"             timestamptz NOT NULL DEFAULT now(),
  "updatedAt"             timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE "verification" (
  "id"         text PRIMARY KEY,
  "identifier" text NOT NULL,
  "value"      text NOT NULL,
  "expiresAt"  timestamptz NOT NULL,
  "createdAt"  timestamptz NOT NULL DEFAULT now(),
  "updatedAt"  timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX "session_userId_idx" ON "session" ("userId");
CREATE INDEX "account_userId_idx" ON "account" ("userId");
CREATE INDEX "verification_identifier_idx" ON "verification" ("identifier");

-- Down Migration
DROP TABLE "verification";
DROP TABLE "account";
DROP TABLE "session";
DROP TABLE "user";
```

## API

`src/auth/auth.ts`
```ts
import { betterAuth } from 'better-auth';
import type { Config } from '../config';
import { createPool } from '../db/pool';

export function createAuth(config: Config) {
  return betterAuth({
    // BetterAuth wants real Date objects, so it gets its own pool without the ISO-string parser.
    database: createPool(config.DATABASE_URL, { dates: 'date' }),
    baseURL: config.APP_URL,
    secret: config.BETTER_AUTH_SECRET,
    trustedOrigins: [config.APP_URL],
    emailAndPassword: { enabled: true },
  });
}
export type Auth = ReturnType<typeof createAuth>;
```

`src/auth/require-auth.ts`
```ts
import { fromNodeHeaders } from 'better-auth/node';
import type { RequestHandler } from 'express';
import { HttpError } from '../http/errors';
import type { Auth } from './auth';

export const requireAuth =
  (auth: Auth): RequestHandler =>
  async (req, res, next) => {
    const session = await auth.api.getSession({ headers: fromNodeHeaders(req.headers) });
    if (!session) throw new HttpError(401, 'Unauthorized');
    res.locals.user = { id: session.user.id, name: session.user.name, email: session.user.email };
    next();
  };
```

`container.ts` creates `auth` and `requireAuth` and returns them alongside the controllers. `app.ts` changes:

```ts
import { toNodeHandler } from 'better-auth/node';

const app = express();
// MUST be mounted before express.json() - BetterAuth reads the raw request body itself.
app.all('/api/auth/*splat', toNodeHandler(container.auth));
app.use(express.json());
...
api.use('/posts', container.requireAuth, postsRoutes(...));
```

Tests: `createApp` takes whatever the container holds, so integration tests build the container and then override `requireAuth` with a stub that sets `res.locals.user = { id: 'user-1' }` after inserting that user row (FK). Add one test that an unauthenticated request gets 401 using the real `requireAuth`.

## Changes to Posts

- Migration: `author_id text NOT NULL REFERENCES "user" ("id") ON DELETE CASCADE` on `posts`, plus an index.
- Shared `Post` gets `authorId: string`.
- Repository `create(authorId, input)` inserts it; `COLUMNS` adds `author_id AS "authorId"`.
- Handlers read `res.locals.user.id` and pass it to the controller. Keep reads open to any signed-in user; `create(userId, input)`, `update(userId, id, input)` and `remove(userId, id)` take `userId`. The controller loads the post first and throws 403 when `post.authorId !== userId`:
  ```ts
  private async mustOwn(userId: string, id: string): Promise<Post> {
    const post = await this.get(id); // 404 if missing
    if (post.authorId !== userId) throw new HttpError(403, 'Not your post');
    return post;
  }
  ```
  Update the unit tests from 02-backend.md accordingly (fake repo now needs `findById` for update/remove; add a 403 non-author test) and the route integration tests (author id = the stubbed user).
- Add a typed `Express.Locals` augmentation (`declare global { namespace Express { interface Locals { user?: { id: string; name: string; email: string } } } }`) so `res.locals.user` is typed.

## Web

- `src/lib/auth-client.ts`: `import { createAuthClient } from 'better-auth/react'; export const authClient = createAuthClient();` (same origin, so the Vite proxy and the Express server both work).
- Screens under `features/auth/`: `login/LoginScreen.tsx` and `signup/SignUpScreen.tsx`, built with `useAppForm`, `TextField` (email, password, name), calling `authClient.signIn.email` / `authClient.signUp.email`, showing `error.message`, navigating to `/posts` on success.
- Routes: `routes/login.tsx`, `routes/signup.tsx` (thin). Move the whole `routes/posts/` folder to `routes/_authed/posts/` and add the layout file `routes/_authed.tsx` (`component: Outlet`). Route ids gain the prefix, so update every `createFileRoute('/_authed/posts/...')` string, `getRouteApi('/_authed/posts/$postId/')`, and redirect/`Link` targets keep using the public paths (`/posts`). Final tree: `routes/{__root,index,login,signup,_authed}.tsx` and `routes/_authed/posts/{index,new}.tsx`, `routes/_authed/posts/$postId/{index,edit}.tsx`. The layout's `beforeLoad` does:
  ```ts
  const { data } = await authClient.getSession();
  if (!data) throw redirect({ to: '/login' });
  return { user: data.user };
  ```
- Header in `__root.tsx`: `authClient.useSession()`; show the user's name in a `Popover` with a "Sign out" button (`authClient.signOut()` then navigate to `/login`), or Login/Sign up links when signed out. Use lucide `LogOut`/`User` icons.
- axios: add a response interceptor that, on 401, sends the user to `/login`.
- Tests: `LoginScreen` renders and shows validation messages.
