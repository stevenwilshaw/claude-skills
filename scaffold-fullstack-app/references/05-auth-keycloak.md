# Auth option B: Keycloak IDP with the BFF pattern

The Express API is the OIDC client ("Backend For Frontend"). It performs the Authorization Code flow with PKCE against Keycloak, keeps the tokens server-side in a Postgres-backed session, and hands the browser only an **httpOnly session cookie**. The SPA never sees a token, so there is nothing for XSS to steal and no token handling in the frontend.

Flow: `GET /api/auth/login` -> redirect to Keycloak (PKCE challenge + state) -> user logs in -> Keycloak redirects to `/api/auth/callback` -> server exchanges code (+ verifier + client secret), stores tokens in the session, sets the cookie -> redirects to the SPA.

The BFF is a confidential client, so it uses PKCE **and** a client secret; PKCE protects the code exchange even if the code leaks, the secret authenticates the BFF itself.

## Packages

- api: `openid-client` (v6 API), `express-session`, `connect-pg-simple`; dev: `@types/express-session`, `@types/connect-pg-simple`
- web: nothing extra

## Env (append to `.env.template`, extend `config.ts`)

```dotenv
# --- Auth (Keycloak BFF) ---
APP_URL=http://localhost:5173
KEYCLOAK_PORT=8080
OIDC_ISSUER=http://localhost:8080/realms/app
OIDC_CLIENT_ID=app-web
OIDC_CLIENT_SECRET=dev-client-secret
SESSION_SECRET=dev-only-change-me-dev-only-change-me
```
`config.ts`: add `APP_URL`, `OIDC_ISSUER`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, `SESSION_SECRET` (min 32), and `OIDC_ALLOW_HTTP: z.enum(['true', 'false']).default('false').transform((v) => v === 'true')`. Tests need a `testConfig()` helper that supplies all of these (plus any S3 vars) instead of the bare `DATABASE_URL` shown in 02-backend.md.

Running the production image locally against the dev stack: the container needs `OIDC_ISSUER`, `DATABASE_URL` (and `S3_ENDPOINT`) reachable from inside it - use `--network <project>_default` with service hostnames (`db`, `keycloak`, `minio`) - plus `-e APP_URL=http://localhost:3000 -e OIDC_ALLOW_HTTP=true` and the client/session secrets. For the browser login to work the issuer hostname must also resolve for the browser, so for a pure smoke test just check `/`, `/api/health` and that `/api/posts` returns 401. Derive `cookieSecure = APP_URL.startsWith('https://')` in code (so plain-http local Docker runs still get a cookie).

## docker-compose: add Keycloak

```yaml
  keycloak:
    image: quay.io/keycloak/keycloak:26.0
    command: start-dev --import-realm
    environment:
      KC_BOOTSTRAP_ADMIN_USERNAME: admin
      KC_BOOTSTRAP_ADMIN_PASSWORD: admin
      KC_HEALTH_ENABLED: "true"
    ports:
      - "${KEYCLOAK_PORT:-8080}:8080"
    volumes:
      - ./docker/keycloak/realm.json:/opt/keycloak/data/import/realm.json:ro
    # The image has no curl; probe the management port (9000) with bash's /dev/tcp.
    # Without a healthcheck `--wait` returns before the realm exists and API startup (OIDC discovery) races it.
    healthcheck:
      test: ["CMD-SHELL", "exec 3<>/dev/tcp/127.0.0.1/9000 && printf 'GET /health/ready HTTP/1.1\\r\\nHost: localhost\\r\\nConnection: close\\r\\n\\r\\n' >&3 && grep -q UP <&3"]
      interval: 5s
      timeout: 5s
      retries: 40
```
No persistence volume: the dev realm is re-imported from `realm.json` on every start, and mounting a named volume at `/opt/keycloak/data/h2` crashes Keycloak (root-owned volume, Keycloak runs as UID 1000). If persistence is ever wanted, mount at `/opt/keycloak/data`. (Verify the image tag exists; use the newest 26.x.) `start-dev` is for local development only - say so in the README.

`docker/keycloak/realm.json` - a pre-configured realm so it works on first boot:
```json
{
  "realm": "app",
  "enabled": true,
  "sslRequired": "none",
  "registrationAllowed": false,
  "loginWithEmailAllowed": true,
  "clients": [
    {
      "clientId": "app-web",
      "name": "App (BFF)",
      "enabled": true,
      "protocol": "openid-connect",
      "publicClient": false,
      "secret": "dev-client-secret",
      "standardFlowEnabled": true,
      "directAccessGrantsEnabled": false,
      "implicitFlowEnabled": false,
      "redirectUris": ["http://localhost:5173/api/auth/callback", "http://localhost:3000/api/auth/callback"],
      "webOrigins": ["+"],
      "attributes": {
        "pkce.code.challenge.method": "S256",
        "post.logout.redirect.uris": "http://localhost:5173/*##http://localhost:3000/*"
      }
    }
  ],
  "users": [
    {
      "username": "dev",
      "email": "dev@example.com",
      "firstName": "Dev",
      "lastName": "User",
      "enabled": true,
      "emailVerified": true,
      "credentials": [{ "type": "password", "value": "dev", "temporary": false }]
    }
  ]
}
```
Document the dev login (`dev` / `dev`) and Keycloak admin console (`admin` / `admin` at `http://localhost:8080`) in the README. Mention that for production the issuer, secret and redirect URIs must be changed.

## Migration - session store (`create-sessions`)

```sql
-- Up Migration
CREATE TABLE "sessions" (
  "sid"    varchar NOT NULL PRIMARY KEY,
  "sess"   json NOT NULL,
  "expire" timestamp(6) NOT NULL
);
CREATE INDEX "sessions_expire_idx" ON "sessions" ("expire");

-- Down Migration
DROP TABLE "sessions";
```
Use `tableName: 'sessions'` and `createTableIfMissing: false` in connect-pg-simple (migrations own the schema).

## API

`src/auth/oidc.ts` - discovery happens once at startup:
```ts
import * as client from 'openid-client';
import type { Config } from '../config';

export async function createOidc(config: Config) {
  return client.discovery(new URL(config.OIDC_ISSUER), config.OIDC_CLIENT_ID, config.OIDC_CLIENT_SECRET, undefined, {
    // Keycloak dev runs on plain http. Only allowed outside production, or when explicitly opted in
    // (needed to run the production Docker image locally against the dev Keycloak).
    execute: config.NODE_ENV === 'production' && !config.OIDC_ALLOW_HTTP ? [] : [client.allowInsecureRequests],
  });
}
export type Oidc = Awaited<ReturnType<typeof createOidc>>;
```
Because discovery is async, make `buildContainer` async (`await createOidc(config)`) and `await` it in `index.ts`; tests for the posts feature use a stubbed `requireAuth` and skip OIDC entirely.

`src/auth/session.ts`
```ts
import PgSession from 'connect-pg-simple';
import session from 'express-session';
import type { Config } from '../config';
import type { Db } from '../db/pool';

const Store = PgSession(session);

export const sessionMiddleware = (config: Config, db: Db) =>
  session({
    name: 'sid',
    store: new Store({ pool: db, tableName: 'sessions', createTableIfMissing: false }),
    secret: config.SESSION_SECRET,
    resave: false,
    saveUninitialized: false, // only create a session once someone logs in / starts the flow
    cookie: {
      httpOnly: true,
      sameSite: 'lax', // must stay lax (not strict): the OIDC redirect back from Keycloak is a cross-site navigation
      secure: config.APP_URL.startsWith('https://'),
      maxAge: 1000 * 60 * 60 * 8,
    },
  });

declare module 'express-session' {
  interface SessionData {
    oidc?: { codeVerifier: string; state: string };
    user?: { sub: string; name?: string; email?: string };
    tokens?: { accessToken: string; refreshToken?: string; idToken?: string; expiresAt: number };
  }
}
```
Add `app.set('trust proxy', 1)` when `NODE_ENV === 'production'` so secure cookies work behind a TLS-terminating proxy.

`src/auth/auth.routes.ts` + `auth.handlers.ts` + `auth.controller.ts` follow the same feature split as everything else (routes -> handlers -> controller class with `Oidc` and `Config` injected). Logic:

```ts
// controller (Express-free except it receives the session object)
async startLogin(sess: SessionData): Promise<string> {
  const codeVerifier = client.randomPKCECodeVerifier();
  const state = client.randomState();
  sess.oidc = { codeVerifier, state };
  const url = client.buildAuthorizationUrl(this.oidc, {
    redirect_uri: `${this.config.APP_URL}/api/auth/callback`,
    scope: 'openid profile email', // do NOT add offline_access: Keycloak rejects it for this client/user ("Offline tokens not allowed") and login fails. Normal refresh tokens are still issued.
    code_challenge: await client.calculatePKCECodeChallenge(codeVerifier),
    code_challenge_method: 'S256',
    state,
  });
  return url.href;
}

async finishLogin(sess: SessionData, callbackUrl: URL) {
  if (!sess.oidc) throw new HttpError(400, 'Login not started');
  const tokens = await client.authorizationCodeGrant(this.oidc, callbackUrl, {
    pkceCodeVerifier: sess.oidc.codeVerifier,
    expectedState: sess.oidc.state,
  });
  const claims = tokens.claims()!;
  return { user: { sub: claims.sub, name: claims.name as string | undefined, email: claims.email as string | undefined }, tokens: toStored(tokens) };
}
```
Handlers:
- `GET /api/auth/login` -> `res.redirect(await controller.startLogin(req.session))`; save the session before redirecting (`req.session.save`).
- `GET /api/auth/callback` -> build `new URL(config.APP_URL + req.originalUrl)`; call `finishLogin`; **regenerate the session** (`req.session.regenerate`, promisified) to prevent fixation, then store `user`/`tokens`, then `redirect(APP_URL + '/')`.
- `GET /api/auth/me` -> `{ user }` or 401 JSON.
- `POST /api/auth/logout` -> build `client.buildEndSessionUrl(oidc, { post_logout_redirect_uri: APP_URL, id_token_hint })`, destroy the session, clear cookie `sid`, return `{ logoutUrl }`. The SPA then sets `window.location.href = logoutUrl`. POST (not GET) so a cross-site image tag cannot log users out.

`src/auth/require-auth.ts` - loads `req.session.user`, refreshes tokens when close to expiry, otherwise 401:
```ts
export const requireAuth = (oidc: Oidc): RequestHandler => async (req, res, next) => {
  const { user, tokens } = req.session;
  if (!user || !tokens) throw new HttpError(401, 'Unauthorized');
  if (tokens.expiresAt - Date.now() < 30_000) {
    if (!tokens.refreshToken) { await destroy(req); throw new HttpError(401, 'Session expired'); }
    try {
      const fresh = await client.refreshTokenGrant(oidc, tokens.refreshToken);
      req.session.tokens = toStored(fresh, tokens); // keep the old refresh/id token if the response omits them
    } catch {
      await destroy(req);
      throw new HttpError(401, 'Session expired');
    }
  }
  res.locals.user = { id: user.sub, name: user.name, email: user.email };
  next();
};
```
`toStored(tokenResponse, previous?)` maps `access_token`, `refresh_token`, `id_token`, and `expiresAt = Date.now() + tokenResponse.expiresIn()! * 1000`.

CSRF: cookies are `SameSite=Lax` and the API only accepts JSON on state-changing routes (forces a CORS preflight cross-origin), which is the baseline. Add a cheap extra guard: a middleware on non-GET `/api` requests requiring the header `X-Requested-With: XMLHttpRequest` (set it as a default header on the axios instance); `POST /api/auth/logout` is covered by it too.

Helpers the snippets rely on (write them in `src/auth/`):
```ts
// promisified session ops - express-session uses callbacks
const save = (req: Request) => new Promise<void>((res, rej) => req.session.save((e) => (e ? rej(e) : res())));
const regenerate = (req: Request) => new Promise<void>((res, rej) => req.session.regenerate((e) => (e ? rej(e) : res())));
const destroy = (req: Request) => new Promise<void>((res, rej) => req.session.destroy((e) => (e ? rej(e) : res())));

import type { TokenEndpointResponse, TokenEndpointResponseHelpers } from 'openid-client';
function toStored(t: TokenEndpointResponse & TokenEndpointResponseHelpers, prev?: SessionData['tokens']) {
  return {
    accessToken: t.access_token,
    refreshToken: t.refresh_token ?? prev?.refreshToken,
    idToken: t.id_token ?? prev?.idToken,
    expiresAt: Date.now() + (t.expiresIn() ?? 300) * 1000, // expiresIn() may be undefined
  };
}
```
The controller's session parameter should be typed `Partial<SessionData>` (`req.session` does not satisfy `SessionData` under strict optional typing).

Container shape for testability: `Container` exposes `requireAuth: RequestHandler` and `authController: AuthController | null`; production wiring (async, does OIDC discovery) fills them, and integration tests build the container by hand with a stub `requireAuth` and `authController: null`, never contacting Keycloak. `createApp(config, container, db)` takes `db` for the session store. Because of the CSRF guard below, integration tests must send `X-Requested-With: XMLHttpRequest` on non-GET requests.

Mount order in `app.ts`: `express.json()` -> `sessionMiddleware` -> CSRF header guard -> `/api/auth` routes -> `api.use('/posts', container.requireAuth, postsRoutes(...))`.

Dev wiring note: the browser talks to the Vite origin (5173) which proxies `/api`, so `redirect_uri` is `http://localhost:5173/api/auth/callback`, the cookie belongs to the 5173 origin, and everything stays same-origin. In the Docker image it is the app's own origin. Keycloak is the only other origin.

## Changes to Posts

- `author_id text NOT NULL` on `posts` (the Keycloak `sub`, no FK - there is no local user table), plus an index.
- Shared `Post` gets `authorId`. Repository inserts/selects it; handlers pass `res.locals.user.id`; controller enforces author-only update/delete (403) exactly like in the BetterAuth variant.
- Typed `Express.Locals` augmentation for `user`.

## Web

- `src/features/auth/auth.queries.ts`: `authQueries.me()` = `queryOptions` calling `GET /auth/me`, mapping a 401 to `null` (`validateStatus` or catch).
- Route guard: a pathless `routes/_authed.tsx` layout; `beforeLoad` runs `context.queryClient.fetchQuery(authQueries.me())` and, if `null`, `throw redirect({ href: '/api/auth/login' })` (a full-page navigation, because the BFF endpoint redirects to Keycloak). Posts routes live under `routes/_authed/` (move `routes/posts/` there; `routes/index.tsx` stays outside). Route ids gain the prefix: `createFileRoute('/_authed/posts/')`, `getRouteApi('/_authed/posts/$postId/')`, `getRouteApi('/_authed/posts/$postId/edit')`; links and redirects keep the public `/posts` paths. Return `{ user }` into route context.
- Header: show the user's name in a `Popover` with "Sign out" -> `POST /auth/logout` -> `window.location.href = data.logoutUrl`.
- axios: default header `X-Requested-With: XMLHttpRequest`; on 401 responses, `window.location.href = '/api/auth/login'`.
- No login/signup screens: Keycloak hosts them.

## Verifying

With compose up: open the SPA, you should be bounced to the Keycloak login, sign in as `dev`/`dev`, land on `/posts`, and `GET /api/auth/me` should show the user. `curl localhost:3000/api/posts` without a cookie must return 401. Check in devtools that no token is visible to JS (`document.cookie` must not contain `sid`).
