# Implementation notes

Sketches for the parts that are easy to get subtly wrong. Adapt names to the repo's conventions, and confirm Octokit option names and GitHub's manifest field names against the installed versions and GitHub's docs ("Registering a GitHub App from a manifest"), since both evolve.

## Contents
- SecretBox (encryption at rest)
- Signed state (two purposes)
- GithubGateway
- Manifest and the two callbacks
- The seam for later features
- Ids

## SecretBox

AES-256-GCM, random 12-byte IV per message, a version prefix so keys can be rotated later without guessing the format. `GITHUB_ENCRYPTION_KEY` is 32 random bytes, base64.

```ts
import { createCipheriv, createDecipheriv, randomBytes } from 'node:crypto';

export class SecretBox {
  private readonly key: Buffer;
  constructor(base64Key: string) {
    this.key = Buffer.from(base64Key, 'base64');
    if (this.key.length !== 32) throw new Error('GITHUB_ENCRYPTION_KEY must be 32 bytes, base64-encoded');
  }
  encrypt(plain: string): string {
    const iv = randomBytes(12);
    const c = createCipheriv('aes-256-gcm', this.key, iv);
    const ct = Buffer.concat([c.update(plain, 'utf8'), c.final()]);
    return ['v1', iv, c.getAuthTag(), ct].map((p) => (typeof p === 'string' ? p : p.toString('base64'))).join('.');
  }
  decrypt(blob: string): string {
    const [v, iv, tag, ct] = blob.split('.');
    if (v !== 'v1' || !iv || !tag || !ct) throw new Error('Unrecognised ciphertext');
    const d = createDecipheriv('aes-256-gcm', this.key, Buffer.from(iv, 'base64'));
    d.setAuthTag(Buffer.from(tag, 'base64'));
    return Buffer.concat([d.update(Buffer.from(ct, 'base64')), d.final()]).toString('utf8');
  }
}
```

Decrypt only at the moment of use (building the App auth); don't keep plaintext keys anywhere but the per-connection client cache.

## Signed state (two purposes)

Stateless: `base64url(json).base64url(hmac)`. Derive a subkey **per purpose** so the encryption key is never an HMAC key and a "create" state can never be replayed as an "install" state.

```ts
import { createHmac, hkdfSync, timingSafeEqual } from 'node:crypto';

const subkey = (master: Buffer, label: string) => Buffer.from(hkdfSync('sha256', master, '', label, 32));

function sign(master: Buffer, label: string, payload: object, ttlMs: number, now = Date.now()) {
  const body = Buffer.from(JSON.stringify({ ...payload, exp: now + ttlMs })).toString('base64url');
  const mac = createHmac('sha256', subkey(master, label)).update(body).digest('base64url');
  return `${body}.${mac}`;
}

function verify(master: Buffer, label: string, state: string, now = Date.now()) {
  const [body, mac, extra] = state.split('.');
  if (!body || !mac || extra !== undefined) return null;
  const expected = createHmac('sha256', subkey(master, label)).update(body).digest();
  const given = Buffer.from(mac, 'base64url');
  if (given.length !== expected.length || !timingSafeEqual(given, expected)) return null;
  try {
    const p = JSON.parse(Buffer.from(body, 'base64url').toString()) as Record<string, unknown> & { exp: number };
    return p.exp > now ? p : null;
  } catch {
    return null;
  }
}

// create: { userId, name, org? }, 15 min     install: { userId, connectionId }, 15 min
export const signCreateState = (m: Buffer, p: { userId: string; name: string; org?: string }) => sign(m, 'github-create', p, 15 * 60_000);
export const signInstallState = (m: Buffer, p: { userId: string; connectionId: string }) => sign(m, 'github-install', p, 15 * 60_000);
// verifyCreateState / verifyInstallState: call verify(), then check each field's type and return a typed object or null.
```

## GithubGateway

All network access to GitHub sits behind one interface, so controller tests use a fake and the real thing is only tested by hand. Everything is `github.com`, so there is no base URL to carry.

```ts
export interface AppCreds { connectionId: string; appId: string; privateKey: string }   // decrypted at point of use
export interface GithubGateway {
  convertManifestCode(code: string): Promise<{ appId: string; appSlug: string; appName: string; privateKey: string }>;
  getInstallation(app: AppCreds, installationId: string): Promise<{ accountLogin: string; accountType: 'User' | 'Organization' | 'Enterprise'; repositorySelection: 'all' | 'selected' }>;
  listInstallationRepos(app: AppCreds, installationId: string): Promise<RemoteRepo[]>;
  installationToken(app: AppCreds, installationId: string, o?: { repositoryIds?: string[] }): Promise<{ token: string; expiresAt: string }>;
}
```

Real implementation notes:

- **Code exchange** is a plain unauthenticated `POST https://api.github.com/app-manifests/{code}/conversions` (`Accept: application/vnd.github+json`). The response has `id`, `slug`, `name` and `pem` (plus a client secret and webhook secret that this design does not store). On a non-2xx, throw an error with **only the status**, never the body: on success the body is the private key.
- **App-level calls** (`apps.getInstallation`): `new Octokit({ authStrategy: createAppAuth, auth: { appId, privateKey } })`; it signs the JWT for `/app/*` routes itself. This call is also the install check: it 404s for an installation that isn't this App's.
- **Installation calls**: same, plus `installationId` in `auth`. `octokit.paginate(octokit.rest.apps.listReposAccessibleToInstallation, { per_page: 100 })` flattens the pages. Cache one Octokit per `(connectionId, installationId)` in a `Map` (`auth-app` caches the 1-hour token inside it) and evict when a connection is deleted or its key changes.
- **Scoped token**: `client.auth({ type: 'installation', installationId, repositoryIds: [id], refresh: true })` returns a one-off token limited to those repos, bypassing the cached all-repos token. Pass ids as numbers to Octokit.
- Normalise a PEM that arrives with literal `\n` sequences before handing it to `auth-app`.
- **Error mapping**: 401/404 on an installation -> `InstallationGoneError` (the controller marks the connection `revoked`); 403 with `x-github-sso` -> an error carrying that header's URL.

## Manifest and the two callbacks

**Start** (`POST /api/github/connections/start`, body `{ name, org? }`). Validate `name` (trimmed, 1-34 chars) and `org` (`^[A-Za-z0-9](?:[A-Za-z0-9-]{0,38})$`) in the zod schema. `base` is `APP_URL` without a trailing slash.

```ts
const state = encodeURIComponent(signCreateState(key, { userId, name, org }));
const manifest = {
  name,                                          // GitHub rejects a clash on its confirmation page; the user can rename there
  url: base,
  redirect_url: `${base}/api/github/connections/created`,
  setup_url: `${base}/api/github/connections/installed`,
  setup_on_update: true,
  public: false,                                 // owned by the creating account, installable only there: right for one App per connection
  default_permissions: { contents: 'write', pull_requests: 'write', metadata: 'read' },
  default_events: [],                            // no webhooks in v1
};
const formUrl = org
  ? `https://github.com/organizations/${encodeURIComponent(org)}/settings/apps/new?state=${state}`
  : `https://github.com/settings/apps/new?state=${state}`;
return { formUrl, manifest };
```

If GitHub insists on a callback URL for an App with no user authorization, add `callback_urls: [base]`; check against the current docs.

**Submit** (SPA): GitHub wants a browser form POST.

```ts
const f = document.createElement('form');
f.method = 'post'; f.action = formUrl;
const i = document.createElement('input');
i.type = 'hidden'; i.name = 'manifest'; i.value = JSON.stringify(manifest);
f.append(i); document.body.append(f); f.submit();
```

**`GET /api/github/connections/created?code&state`**

1. Session required (`requireAuth` ran). `verifyCreateState`; `state.userId` must equal the session user, else redirect `/github?error=invalid_state`. Validate `code` loosely (`^[\w-]{10,100}$`).
2. `convertManifestCode(code)`. The code is single-use and expires after about an hour.
3. Encrypt the PEM and insert the row (`status = 'pending'`, `name` = the name GitHub returns, `app_id`, `app_slug`). If this fails the key is lost: log only a generic message and redirect `/github?error=save_failed`, whose UI text tells the user to delete the App on GitHub and retry.
4. Validate `appSlug` against `^[a-z0-9-]+$`, then redirect to `https://github.com/apps/${appSlug}/installations/new?state=${signInstallState(key, { userId, connectionId })}`.

**`GET /api/github/connections/installed?installation_id&setup_action&state`**

1. Session required. `verifyInstallState`; `state.userId` equals the session user; load the connection through `findAccessible(userId, state.connectionId)`. Any failure -> `/github?error=invalid_state`.
2. `setup_action === 'request'` -> redirect `/github?pending=1` (an org owner must approve; no installation yet).
3. `installation_id` must match `^\d{1,18}$`. Call `getInstallation` with **this connection's** App credentials. A 404 means the id isn't an installation of this App: redirect `/github?error=not_this_app`.
4. Update the row: `installation_id`, `account_login`, `account_type`, `repository_selection`, `status = 'active'`.
5. Redirect `/github?connected=1`. Every redirect target is a fixed path on `APP_URL`, never taken from the query string (no open redirect).

**"Finish on GitHub"** (`GET /connections/:id/install-url`): for a `pending` connection only, return the same install URL with a fresh install state.

## The seam for later features

```ts
async tokenForRepo(userId: string, connectionId: string, githubRepoId: string) {
  const conn = await this.connections.findAccessible(userId, connectionId);   // null if not theirs -> 404
  if (!conn) throw new HttpError(404, 'Connection not found');
  if (conn.status !== 'active') throw new HttpError(409, 'GitHub connection needs to be reconnected');
  try {
    return await this.gateway.installationToken(this.creds(conn), conn.installationId!, { repositoryIds: [githubRepoId] });
  } catch (e) {
    if (e instanceof InstallationGoneError) {
      await this.connections.markRevoked(conn.id);
      throw new HttpError(409, 'GitHub connection needs to be reconnected');
    }
    throw e;
  }
}
```

`findAccessible` is the single place that decides who may use a connection. Today it is `WHERE id = $1 AND user_id = $2`; sharing connections later changes only this query.

## Ids

`pg` returns `bigint` columns as strings, so decide once how ids cross each boundary instead of letting the type drift. Keep GitHub ids (app, installation, repo) as **strings** in DTOs, request bodies (`z.string().regex(/^\d{1,18}$/)`) and the repository layer, and convert to numbers only at the Octokit call. Select `app_id::text` / `installation_id::text` (or alias them) so the repository layer never sees a mixed type.
