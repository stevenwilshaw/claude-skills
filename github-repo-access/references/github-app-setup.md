# GitHub App settings reference

Nobody creates the App by hand: the app creates one per connection through GitHub's App Manifest flow (`references/implementation.md`). This page is for **debugging** (GitHub rejected the manifest, or an installation behaves unexpectedly) and for explaining to the user what the App they are approving can do. Confirm field names against GitHub's "Registering a GitHub App from a manifest" docs, as they change.

## What the manifest requests

| Setting | Value | Why |
|---|---|---|
| Homepage URL | `APP_URL` | Required by GitHub |
| Redirect URL | `{APP_URL}/api/github/connections/created` | Where GitHub sends the browser, with a one-time `code`, after the user confirms creating the App |
| Setup URL | `{APP_URL}/api/github/connections/installed` | Where GitHub sends the browser after the App is installed, with `installation_id` |
| Redirect on update | on | Changing which repos the installation sees comes back to the app and refreshes the connection |
| Public | **off** | The App is owned by the account that created it and installable only there. Right for one App per connection |
| Webhooks | off (v1) | The app reconciles lazily when it uses a connection. Turn on `installation` and `installation_repositories` events later for live sync |
| User authorization (OAuth) | none | Not needed: the install is verified with the App's own credentials |

`{APP_URL}` is the browser-facing origin (`APP_URL` in `.env`). GitHub only has to redirect the *user's browser* there, so `http://localhost:5173` works in dev.

## Permissions (all requested up front)

Repository -> **Contents: Read and write**, **Pull requests: Read and write**, **Metadata: Read-only** (mandatory).

Write access is requested now even though connections only read. Raising permissions later forces the installation owner to re-approve, which on an organisation means asking an owner. The permission is harmless until a feature uses it, and tokens from `tokenForRepo` are scoped to one repo.

## Where things live on GitHub

- **The App**: `https://github.com/settings/apps/<slug>` for an App created on a personal account, or `https://github.com/organizations/<org>/settings/apps/<slug>` for an org. Deleting it there breaks the connection (the app will mark it `revoked` when GitHub answers 401/404).
- **The installation** (which repos the App can see): `https://github.com/settings/installations/<installation_id>`, or `.../organizations/<org>/settings/installations/<installation_id>` for an org. "Manage on GitHub" in the UI links here.

## Common problems

- **GitHub says the name is taken**: App names are unique across all of GitHub. The user renames it on the confirmation page.
- **Creating under an organisation**: only org owners (or members allowed to manage Apps) can. GitHub shows the error on its page.
- **`setup_action=request` on install**: an org owner has to approve the installation. The connection stays `pending` until they do and the user comes back.
- **Enterprise Cloud with SAML SSO**: if GitHub answers a call with 403 and an `X-GitHub-SSO` header, relay that URL to the user so they can authorise the App for their SSO session.
- **Redirect goes to the wrong address**: `APP_URL` must be the origin the browser really uses (scheme, host and port).
- **Not supported**: GitHub Enterprise Server on its own domain. Everything is hardcoded to `github.com`; adding it later means `web_url`/`api_url` columns on the connection and threading them through the gateway.
