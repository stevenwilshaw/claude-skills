---
name: github-repo-access
description: Add GitHub connections to this full-stack app, the way Dokploy or Netlify do. A signed-in user adds a connection by naming it; the app creates a dedicated GitHub App for it through GitHub's App Manifest flow, the user approves on GitHub, picks which repos it may see, and lands back in the app with everything connected. One table holds the connection, its encrypted App credentials and its installation. Also covers the repo-listing API, the connections UI, and the server-side service later features (projects, tasks, PRs) use to get a repo-scoped token. Use whenever the user wants to connect GitHub, add a git provider or GitHub integration, link or import repositories, create a GitHub App from the app, or decide how to store GitHub credentials - including when they only describe the end goal (e.g. "users should be able to run tasks and open PRs on their own repos"), since this is the access layer that has to exist first. Not for "Sign in with GitHub" login, Keycloak/SSO identity brokering, GitHub Actions or CI config, other providers like GitLab, debugging webhooks, GitHub Enterprise Server on its own domain, or the project, task-running and PR-creation logic itself.
---

# GitHub connections

Lets a user who is already logged in (BetterAuth or Keycloak) **add a GitHub connection**. A connection is one GitHub App, created by this app on the user's behalf, plus its installation on the repos the user chose. That is the whole scope. Projects that link to repos, running tasks and opening PRs are separate features that *consume* connections through one service seam (`GithubAccessService`, Step 4).

## The model: one connection = one App = one row

This is the Dokploy "git provider" model, and it is deliberately simple:

- The user clicks **Add connection**, which opens a modal where they give it a name (pre-filled with a suggestion). That name becomes the GitHub App's name.
- The app sends the browser to GitHub with a ready-made App definition (GitHub's **App Manifest flow**). The user confirms once, GitHub redirects back with a one-time code, and the server exchanges it for the App's credentials.
- The app immediately sends the user to GitHub's install page, where they choose **which repos the App may see**. GitHub redirects back and the connection is active.
- GitHub has two approvals (create the App, install the App). They are chained so it feels like one process, but the user does see two GitHub screens.

Why one App per connection rather than one App shared by everyone: there is no "host" or "app registry" concept, no shared secrets, no per-user ownership proof, and a connection can later be owned by one user or shared without changing anything about how it works. The cost is a GitHub App per connection (names must be unique on GitHub) and the extra confirmation click. The App is **private** (owned by the account that created it, installable only there), which is right for this model.

What is and isn't stored: only the App's **private key**, encrypted. Installation tokens (1 hour) are minted on demand and never stored. There is no OAuth, no client secret and no per-user GitHub token, so a leaked row can't be used to call GitHub without the encryption key.

**Login is unrelated.** Keycloak/BetterAuth identify the user; a GitHub connection is separate data. Don't reuse BetterAuth's `account` table or "Sign in with GitHub": the Keycloak variant has no such table, and a login identity is not a repo-access identity.

Hardcoded to `github.com` (including Enterprise Cloud orgs, which live there). GitHub Enterprise Server on its own domain is out of scope; if it's needed later, add `web_url`/`api_url` columns and thread them through the gateway.

## Step 1 - Inspect before writing

Read `CLAUDE.md` and follow its architecture rules (raw SQL with `pg`, no ORM, manual wiring in `container.ts`, features in `apps/api/src/features/<name>/` with routes/handlers/controller/repository, shared zod schemas in `packages/shared`, TanStack Query on the web). Use `features/posts` as the template for file shape and tests. Confirm:

- Which auth is in place (`apps/api/src/auth/`). The only thing that varies is where the user id comes from; both variants expose it via `requireAuth` -> `res.locals.user.id`.
- The latest migration filename. Create new ones with `npm run migrate:create -- <name>`; never edit an applied migration.
- Whether any `github*` tables, routes or env vars already exist (for example from an earlier "hosts" based version). Replace them with a new migration rather than layering on top.
- That a shared `Modal` component exists in `src/components` (controlled: `open`, `onOpenChange`, `title`, optional `description`, `children`). The skill assumes one does, because the add-connection form lives in a modal. If there isn't one, say so and ask before inventing a different UI.

## Step 2 - Ask what the code can't tell you

One batch, before generating anything:

1. **Who can use a connection?** Default: only the user who created it (`user_id` is the owner). If connections should be shared (a team, a workspace), ask how sharing is modelled; the table doesn't change, only the access check in `findAccessible` (Step 4).
2. **Does a "projects" (or similar) feature already exist** that will link to repos? If so, it will call `GithubAccessService`; don't create a repositories table. Repos are listed live from GitHub and the project stores `connectionId` + `githubRepoId`.
3. **Any naming rules for the App?** Default: the form is **pre-filled with a suggested name** built from the app's hostname plus a few random characters (so the user can just click through, and clashes on GitHub are unlikely), and the user can edit it. Names are 1-34 characters and must be unique on GitHub; GitHub rejects clashes on its confirmation page. Offer a fixed prefix instead if they want a convention.

## Step 3 - Data model

One table, in the style of `create-posts` (snake_case, uuid ids, `timestamptz`):

```sql
-- Up Migration
CREATE TABLE github_connections (
  id                   uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id              text NOT NULL REFERENCES "user" ("id") ON DELETE CASCADE,  -- owner (see Step 2)
  name                 text NOT NULL,                  -- the GitHub App's name, as GitHub saved it
  app_id               bigint NOT NULL UNIQUE,
  app_slug             text NOT NULL,
  private_key_enc      text NOT NULL,                  -- SecretBox ciphertext (PEM)
  status               text NOT NULL DEFAULT 'pending'
                         CHECK (status IN ('pending', 'active', 'suspended', 'revoked')),
  installation_id      bigint,                         -- NULL until the App is installed
  account_login        text,
  account_type         text CHECK (account_type IN ('User', 'Organization', 'Enterprise')),
  repository_selection text CHECK (repository_selection IN ('all', 'selected')),
  created_at           timestamptz NOT NULL DEFAULT now(),
  updated_at           timestamptz NOT NULL DEFAULT now(),
  CHECK (status = 'pending' OR installation_id IS NOT NULL)
);
CREATE INDEX github_connections_user_id_idx ON github_connections (user_id);

-- Down Migration
DROP TABLE github_connections;
```

`pending` means the App exists on GitHub but isn't installed yet (the user abandoned the install step); the UI offers "Finish on GitHub". `revoked` means GitHub says the installation is gone. If an earlier version created `github_hosts` / `github_repositories`, drop them in the same migration.

## Step 4 - Backend

New feature folder `apps/api/src/features/github/` following the posts template. Read `references/implementation.md` for code of the tricky parts (SecretBox, signed state, gateway, both callbacks, the seam). Add `GITHUB_ENCRYPTION_KEY` (32 bytes, base64) to `config.ts` and `.env.template`; add `@octokit/rest` and `@octokit/auth-app` to `apps/api`.

Endpoints, all behind `requireAuth` and registered in `container.ts` and `app.ts`:

| Method + path | Purpose |
|---|---|
| `POST /api/github/connections/start` `{name, org?}` | Returns the App manifest, GitHub's form URL and a signed `state`; the SPA auto-submits it to GitHub. `org` creates the App under an organisation instead of the user's account |
| `GET /api/github/connections/created?code&state` | GitHub redirects here after the user confirms. Exchanges the code, saves a `pending` connection, redirects the browser to the install page |
| `GET /api/github/connections/installed?installation_id&setup_action&state` | GitHub redirects here after install. Verifies, marks the connection `active`, redirects to the SPA |
| `GET /api/github/connections` | The user's connections (never secrets) |
| `GET /api/github/connections/:id/install-url` | Fresh install URL for a `pending` connection ("Finish on GitHub") |
| `DELETE /api/github/connections/:id` | Removes our row. The App and installation stay on GitHub; the UI links there |
| `GET /api/github/connections/:id/repositories?q&page` | Live list of repos the installation can see (what the projects feature picks from) |

Rules that matter and why:

- **The two redirects are browser GETs carrying the user's session.** Both verify a **signed, expiring `state`** bound to the user (and to the connection id on the install redirect), and require the session user to match. That stops CSRF and cross-user linking. Stateless HMAC means no extra table. Use different keys per purpose so a "create" state can never be replayed as an "install" state.
- **No ownership proof is needed on install.** The App belongs to this one connection, so calling `GET /app/installations/{id}` with *this connection's* App credentials only succeeds for an installation of this App; a forged `installation_id` gets a 404. That replaces the OAuth round trip an app-shared-by-everyone design needs.
- **The private key is shown once**, in the code exchange. Encrypt it before it touches the database, never log the response, the code or any decrypted value, and if the insert fails tell the user to delete the App on GitHub and try again (the key can't be fetched again).
- **Manifest settings** (full block in `references/implementation.md`): `public: false`, the Contents, Pull requests and Metadata permissions requested up front (raising permissions later forces the installation owner to re-approve), no webhooks, no OAuth. Redirect targets are fixed paths on `APP_URL`; the install redirect goes to `github.com/apps/<slug>/...` with the slug validated (`^[a-z0-9-]+$`), never to anything from the query string.
- **Every connection route checks ownership** through one repository method (`findAccessible(userId, id)`) and answers 404, not 403, for someone else's connection. Sharing connections later means changing that one method.
- **`setup_action=request`** means an org owner must approve; there is no installation yet. Show "waiting for approval", not an error.
- **Mark a connection `revoked`** when GitHub returns 401/404 for its installation, so the UI asks to reconnect instead of failing mysteriously. If GitHub answers 403 with an `X-GitHub-SSO` header (Enterprise Cloud with SAML), return its URL to the user.
- **Secrets never leave the server.** DTOs in `packages/shared` have no secret fields by construction.

**The seam for later features** (projects, tasks, PRs) - `GithubAccessService`, unit-testable behind `GithubGateway`:

```ts
interface GithubAccessService {
  listRepositories(userId: string, connectionId: string, o: { q: string; page: number }): Promise<...>;
  // 404 unless the connection is the user's; 409 unless it is active.
  tokenForRepo(userId: string, connectionId: string, githubRepoId: string): Promise<{ token: string; expiresAt: string }>;
  clientForRepo(userId: string, connectionId: string, githubRepoId: string): Promise<Octokit>;
}
```

`tokenForRepo` requests an installation token **scoped to that single repository** (`repositoryIds: [githubRepoId]`), so a task that clones one repo can't touch the others even if the token leaks, and GitHub refuses repos the installation can't see. Document the seam in `CLAUDE.md` so other features use it instead of reaching into the table.

## Step 5 - Web

Per `CLAUDE.md`, routes are thin and UI lives in `src/features/github/<screen>/`:

- `routes/_authed/github/index.tsx` -> `connections/ConnectionsScreen`: the user's connections with a status badge (`pending`, `active`, `revoked`) and these actions: **Add connection** (a header button that opens a modal, below), **Finish on GitHub** for `pending`, **Manage on GitHub** for `active` (`https://github.com/settings/installations/{id}`, or `.../organizations/{org}/settings/installations/{id}` for an org) to change which repos it can see, **Reconnect** for `revoked`, and **Delete**. The redirects land back here with `?connected=1`, `?pending=1` or `?error=<code>`.
- **Add connection** is a button in the screen header that opens the shared `Modal` containing the form (name, optional organisation, Cancel and Create GitHub App buttons), not an inline form. Keep the `Modal`'s state in the screen (`adding`), and show start errors inside the modal; on success the browser leaves for GitHub, so there is nothing to close.
- The name field starts with a suggestion, regenerated each time the modal opens (so a second connection doesn't reuse the first's, and the organisation and any old error are reset too): `window.location.hostname` with runs of non-alphanumerics turned into hyphens, trimmed, capped at **27 characters** (fallback `app`), then `-` and 6 random base-36 characters from `crypto.getRandomValues`. That is at most 34 characters, GitHub's limit (`localhost-a1b2c3`, `app-example-com-x9k2m1`).
- Add connection calls `connections/start`, then builds a hidden `<form method="post">` to GitHub's URL with the manifest in a `manifest` field and submits it (GitHub takes a browser form POST, not an API call). Code is in `references/implementation.md`.
- A small read-only "Repositories" list per active connection (from `.../repositories`) lets the user confirm the right repos are visible. The repo *picker* for linking projects belongs to the projects feature, which uses the same endpoint.
- Query factories in `github.queries.ts`; invalidate the connections key after delete. Reuse `src/components`; no new UI library.

## Step 6 - Tests and verification

Follow posts: `*.controller.test.ts` with a fake repository and a fake `GithubGateway`. Cover: `created` rejects a bad, expired, other-user or wrong-purpose state and a malformed code; the saved row has the private key encrypted and nothing secret in logs; `installed` rejects a state for another user or connection, and an installation GitHub doesn't recognise for this App; another user's connection gets 404 everywhere; a non-active connection gets 409 from the seam; a gone installation marks the connection revoked. `*.repository.int.test.ts` against real Postgres (insert the user row first - FK). A `secret-box.test.ts` round trip plus tamper and wrong-key failure. Then `npm run typecheck`, `npm test`, `npm run test:int`.

The real GitHub round trip cannot be automated. Say so plainly in the hand-off instead of implying it was verified; GitHub's manifest field names should be checked against its "Registering a GitHub App from a manifest" docs, and the manual smoke test is: add a connection, approve twice on GitHub, see the repos list, delete.

## Step 7 - Hand-off

Tell the user what to do: set `GITHUB_ENCRYPTION_KEY`, run `npm run migrate`, open the GitHub page and add a connection. `APP_URL` must be the address their browser really uses, because GitHub redirects back to it (`http://localhost:5173` works in dev). Mention what is deliberately left out so it isn't a surprise: webhooks for live sync (v1 reconciles lazily on use), GitHub Enterprise Server on its own domain, and the projects/task/PR work itself.
