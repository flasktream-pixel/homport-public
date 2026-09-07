# Cloudflare OAuth for desktop apps: what actually works

> This document **stands on its own** so it can be carried into another project. Every fact in it was exercised for real against a real Cloudflare account (2026-09-03 to 2026-09-04) with a WPF app on Windows — none of it is inferred from Cloudflare's documentation. Anything that could not be exercised for real is marked **[UNVERIFIED]**. There are no tokens, no secrets and no full ids in this file.
>
> What differs from the Cloudflare OAuth flow for a **server-side web app**, which is what most documentation describes: there the client can keep a `client_secret`. Here the client is **public** and runs on the user's machine, so there is **no secret**, the redirect goes to **loopback**, and there is **no refresh token**. Those three differences drive almost everything else below.

---

## 0. One-minute summary

| Question | Verified answer |
|---|---|
| Does Cloudflare let a desktop app sign in with OAuth? | **Yes.** Self-managed OAuth client, public client, PKCE S256, no secret. |
| Where does it redirect? | `http://localhost:<FIXED PORT>/callback`. The port **cannot be random**, because Cloudflare matches the registered string verbatim. |
| Is there a refresh token? | **No.** The access token lives exactly **1 hour** (`expires_in` 3599–3600). The client is not allowed to ask for `offline_access`. |
| What happens when it expires? | The user has to sign in again through the browser **and click Authorize again** (Cloudflare does not remember the previous consent for a Private client). |
| Do you have to send `scope`? | **Required.** Leave it out and the consent screen shows "0 total permissions" and will not let you Authorize. |
| Where do the scope identifiers come from? | The client's "scopes" popup on the dashboard, or `GET /client/v4/oauth/scopes`. **They cannot be derived from the display label.** |
| Does the token work against the real API? | **Yes**, as a Bearer token on `api.cloudflare.com`, within the scopes that were granted. |
| Biggest traps? | (1) two Access permission scopes have near-identical identifiers with swapped roles; (2) a `Private` client only lets members of the owning account click consent, and the wrong browser account gets a "private application" message; (3) the failures only surface in a real run — unit tests do not see them. |

---

## 1. Registering the client on the dashboard

Route: `dash.cloudflare.com` → pick the account → **Manage Account** → **OAuth clients** → **Create**.

| Field | What to put in | Why |
|---|---|---|
| Name | The app name | Appears on the consent screen: "X wants to access your account" |
| Client URL | The product's real domain (e.g. `https://example.com`) | Put `http://localhost:...` here and the client still works in **Private** mode, but the dashboard reports `Client URL required` and **you cannot switch it to Public** |
| Redirect URI | `http://localhost:47823/callback` (port is your choice, but **fixed**) | Matched **verbatim** at authorize time and at token exchange. See section 4 |
| Response type / Grant | Code / Authorization Code | |
| Token Authentication Method | The dashboard shows `Client Secret POST` with the secret field greyed out | **Ignore it.** A PKCE token exchange **needs no secret** and still returns 200. Do not embed a secret in a desktop app |
| Scopes | Tick exactly the permissions the app needs, all **Required** | The client **may only request the scopes it registered**; anything else is `invalid_scope` right at authorize |
| Visibility | Private during development | Private = **only members of the account that owns the client** can click Authorize. To let outsiders in: Public + Verified, see section 9 |

Once it is created, copy the **client_id** (32 hex). It is **not a secret**: it sits in plain view inside the authorize URL the user can read in the address bar. Hardcoding it in the app source is the right call; do not read it from a config file the user can edit.

**Getting the scope identifiers**: in the OAuth clients table, click the client's scope count to open the popup — the identifier sits next to the label. Or call `GET https://api.cloudflare.com/client/v4/oauth/scopes` with any API token (returns ~383 scopes). The ones used here:

| Dashboard label | Identifier sent in `scope` |
|---|---|
| DNS Write | `dns.write` |
| Zone Read | `zone.read` |
| Access: Apps and Policies Write | `zone-access.write` |
| Access: Organizations, Identity Providers, and Groups Write | `access-acct.write` |
| Cloudflare Tunnel Write | `argotunnel.write` |
| Account Settings Read | `account-settings.read` |

Three of these cannot be guessed from the label: Tunnel carries its old name `argotunnel`; of the two Access scopes, the one with `zone` in its name works on apps and policies, while the one with `acct` works on organizations and identity providers. Mapping them the wrong way round once cost half a day (section 10).

---

## 2. Endpoints (confirmed via OIDC discovery at `https://dash.cloudflare.com/.well-known/openid-configuration`)

| Purpose | URL |
|---|---|
| Authorize | `https://dash.cloudflare.com/oauth2/auth` |
| Token | `https://dash.cloudflare.com/oauth2/token` |
| Revoke | `https://dash.cloudflare.com/oauth2/revoke` **[UNVERIFIED]** |
| Device authorization | `https://dash.cloudflare.com/oauth2/device/auth` (exists, unused) |
| Userinfo / JWKS | `/oauth2/userinfo`, `/.well-known/jwks.json` (exist, unused) |
| API | `https://api.cloudflare.com/client/v4/...` with header `Authorization: Bearer <access_token>` |

Discovery advertises `token_endpoint_auth_methods_supported` including `none` (public clients are legal), `code_challenge_methods_supported` including `S256`, and `refresh_token` under `grant_types_supported` — but see section 5: the client cannot request `offline_access`, so a refresh token never reaches it.

---

## 3. The full HTTP sequence (language-independent)

```
[1] Generate at random:
    state          = base64url(32 bytes)
    code_verifier  = base64url(64 bytes)
    code_challenge = base64url(SHA256(code_verifier))

[2] Open the loopback listener BEFORE opening the browser (section 4), then open the system browser at:
    https://dash.cloudflare.com/oauth2/auth
      ?client_id=<client_id>
      &redirect_uri=http://localhost:47823/callback      (verbatim, exactly as registered)
      &response_type=code
      &state=<state>
      &code_challenge=<code_challenge>
      &code_challenge_method=S256
      &scope=dns.write zone.read ...                        (REQUIRED, space-separated)

[3] The user clicks Authorize. The browser comes back to:
    GET http://localhost:47823/callback?code=cfoac_...&state=<state>
    On failure:  ?error=access_denied            (user clicked Cancel)
                 ?error=invalid_scope&error_description=...   (asked for an unregistered scope)
    Listener: compare state, accept exactly ONE request, serve a "You can close this tab" page, close the listener.

[4] Exchange the code for a token (POST form-urlencoded, NO client_secret):
    POST https://dash.cloudflare.com/oauth2/token
      grant_type=authorization_code
      code=<code>
      client_id=<client_id>
      redirect_uri=http://localhost:47823/callback
      code_verifier=<code_verifier>
    Never retry this request: the code is single-use.

[5] Response 200:
    { "access_token": "cfoa...", "token_type": "bearer", "expires_in": 3599, "scope": "dns.write zone.read ..." }
    NO refresh_token. The scope field is the list that was ACTUALLY granted; use it to detect missing permissions.

[6] Use it: Authorization: Bearer <access_token> against api.cloudflare.com. Expires after 1 hour.
```

Observed in practice: the `code` carries the prefix `cfoac_`, the access token carries the prefix `cfoa` and runs ~93 characters. Do not build logic on the prefixes — use them only to recognize the values when redacting logs.

---

## 4. The loopback listener: the non-negotiables

- **Fixed port**, the exact port in the registered redirect URI. RFC 8252's "any available port" recommendation **does not apply** to Cloudflare, because the match is verbatim. Consequence: another program can take the port, so you need a dedicated message ("another app is using port X"), not a generic error.
- **Bind two separate listeners**: `127.0.0.1:<port>` and `[::1]:<port>`. The browser may resolve `localhost` to IPv6 first. Do **not** bind `IPv6Any` with `IPv6Only=false`: that catches both families but opens the port on every network interface — a real security bug that made it into a prototype.
- Accept exactly **one** request with a matching `state`, then close immediately. A request with a different `state` gets the "Sign-in could not be verified" page, and its code is not used.
- Callback wait: **5 minutes**; on expiry, close the listener and report a timeout (distinct from the user clicking Cancel).
- The callback page should be self-contained and load nothing external: `<h2>You can close this tab</h2><p>Go back to the app to continue.</p>`.
- Open the URL with the **system browser** (`ShellExecute`), **not** an embedded webview. The user needs to see the real `dash.cloudflare.com` in the address bar to tell it apart from phishing. Only allow the `http`/`https` schemes when opening.

---

## 5. No refresh token — and how to live with it

Measured facts (twice, identical both times):

- `offline_access` is **not in the catalogue of 383 scopes** the dashboard lets you assign to a client. Add it to the `scope` parameter and you get `error=invalid_scope: The OAuth 2.0 Client is not allowed to request scope 'offline_access'`.
- The token response has **no `refresh_token`**. `expires_in` = 3599 or 3600.
- Signing in a second time, same browser, same Cloudflare session: **the consent screen still appears and Authorize still has to be clicked**. There is no silent redirect.

The design we settled on and ran for real:

1. **Separate the credentials that run from the credentials that manage.** Whatever has to keep running (here, the cloudflared connector) uses its own long-lived credential (a tunnel token) stored with DPAPI and **needs no OAuth**. The OAuth access token is only for management operations. When it expires, everything already running keeps running.
2. **No silent refresh, no timer.** Store the access token together with `expires_at`; read it back when the app opens; still valid → `SignedIn`, expired → `SignInRequired`. **Never open a browser by itself at startup.**
3. **Sign in on demand, stitched into the operation.** Write buttons stay clickable in `SignInRequired`; clicking one shows a one-sentence heads-up dialog ("Cloudflare will ask you to allow X again. Click Allow, then come back here."), runs the sign-in, then **resumes exactly the operation that was interrupted**. Wrap it in a function shaped like `RunWriteOperation(closure)`: the closure holds whatever the user typed, and the service handles prompt → sign in → check scopes → check the account is the right one → carry on. The UI has nothing to remember.
4. **Reading needs no token.** Dashboard, status and diagnostics read from local data. A user who opens the app just to see how things are is never asked to sign in.

Real cost: two clicks each time the token has expired and the user wants to *change* something. For an install-once-and-forget app that is acceptable.

**[UNVERIFIED]**: whether a **Public + Verified** client skips the consent screen (many providers do). Whether creating a client through the `POST /oauth/clients` API accepts `offline_access`.

---

## 6. Storing the token and restoring the session

- Store with **DPAPI `CurrentUser`** plus app-specific entropy, one file per secret (`secrets/oauth.access.dpapi`, `secrets/tunnel.token.dpapi`). No Registry, no shared config file.
- The blob holds: the token, `expires_at` (UTC), and the list of granted scopes. **The scopes must be stored too**: after a restore, the preflight needs to know which permissions are already held; forget this and reopening the app reports "missing permissions" even though the token is fine (a bug we actually hit).
- When the app opens: read the blob → still valid → state `SignedIn` and **go straight past the sign-in screen** (a bug we actually hit: Core restored correctly but the UI still opened the Sign in screen).
- Expired blob: leave it on disk (harmless) but do not keep the token string in memory.
- Sign-out: call revoke (**[UNVERIFIED]** on the Cloudflare side), then delete the blob. Sign-out must **not** delete the long-lived credential of whatever is running, if the user only wants to end the management session.

---

## 7. Once you have a token: preflight, and the API errors that mislead

Right after the token exchange, run a cheap preflight to learn which account the token belongs to and whether the permissions are sufficient:

- Use `GET /accounts` (200 with the `account-settings.read` scope). **Do not use `/memberships`**: it demands the extra `memberships.read` scope and returns 403 `code 10000` without it.
- `GET /zones?account.id=<id>` to list domains; only `status` = `active` is usable.
- Compare the token's account with the one stored from last time; on a mismatch, say "You signed in to a different Cloudflare account" rather than writing to the wrong place.

Errors actually hit, and what they really mean:

| Response | What it really means | Commonly misread as |
|---|---|---|
| 403 `code 9999`, message contains `access.api.error.not_enabled` | The account **has Zero Trust turned off**; every `/access/*` call answers this way | Missing scope |
| 403 `code 1010`, `error: auth.forbidden` (note: it sits under the `error` key, not `message`) | Calling an **account-level** endpoint with **zone-level** permission (e.g. `POST /accounts/{id}/access/apps` with `zone-access.write`) | Broken token |
| 403 `code 10000 Authentication error` | No permission at all for that endpoint (e.g. `/memberships`, `/access/tags`) | Zero Trust not enabled |
| 401 | Token expired or revoked | |
| `error=invalid_scope` at authorize | Asked for a scope the client did not register, or for `offline_access` | Network error |

The lesson about **scope level**: the scope named `zone-access.write` only opens `/zones/{zone_id}/access/apps...`; calling `/accounts/{id}/access/apps` needs account-level permission, which this scope does not carry. Zone-level endpoints work with **both** kinds of permission, so choosing zone-level is the safe bet. Knock-on effect: Access app `tags` have to be created first via `POST /accounts/{id}/access/tags` — an endpoint with no zone-level equivalent — so with a zone scope you **cannot use tags**, and the ownership marker has to live in `name`.

---

## 8. Mapping errors onto the UI

| Situation | How to detect it | Message it should carry |
|---|---|---|
| User clicks Cancel / closes the tab | callback `error=access_denied`, or cancelled inside the app | "Sign-in was cancelled." — let them try again |
| No callback within 5 minutes | listener timeout | "We didn't hear back from your browser." — **not** the same as Cancel |
| Wrong `state` / callback with no code | the comparison | "Sign-in could not be verified. Try again." |
| Missing scope after sign-in | compare `scope` in the token response against the required list | "X didn't get the permissions it needs. Sign in again and allow access to your <friendly group>." — the real permission names belong in Details only |
| Loopback port taken | bind fails | "Another app is using the port X needs to finish signing in. Close that app, then try again." |
| Browser will not open | ShellExecute throws | "X couldn't open your browser. Set a default browser in Windows Settings." |
| Wrong browser account with a Private client | Cloudflare shows "This is a private application…" **in the browser**; the app only sees a timeout/cancel | Put it in the instructions: sign in to dash.cloudflare.com with the right account first |
| Token expires mid-flight | 401 on an API call | Move to `SignInRequired` without losing the work in progress |
| Token belongs to another account | preflight account comparison | "You signed in to a different Cloudflare account." |

---

## 9. The road to Public (so outsiders can use it)

Based on the client's current state and one earlier full run through this process for a previous web project:

1. Client URL must be a real domain (`http://localhost` gets `Client URL required`).
2. `logo_uri` must not be empty (missing it blocks you with a `non-empty logo_uri` error).
3. Verify the publisher domain with a DNS TXT record `cloudflare_oauth_client_publisher=<token>`.
4. Visibility Private → Public. Expected result: Visibility Public, Verification Verified.

Before going Public, everyone who tries the app has to be a member of the account that owns the client. That is why a developer should create the client on their own account from the start.

---

## 10. Bugs that only surface in a real run (so as not to repeat them)

The first ten minutes of running for real crashed the app three times, despite **1376 green unit tests**. Written down so that next time we run for real **as soon as the wiring is done**, not at the end:

| Bug | Why the tests missed it | Fix |
|---|---|---|
| The two Access scopes mapped the wrong way round → the API token gate lacked the organization permission | The identifier cannot be derived from the label | Keep the mapping table in one place; tests pin the **identifier**, not the label |
| Default browser signed in to a different account → "private application" ×3 | Not a code bug | Check the context (which account is signed in) before blaming the configuration |
| Dropped `scope` from the URL, assuming Cloudflare would use the configured scopes | No test against the real Cloudflare | `scope` is required, always send it |
| A spinner in a template threw "name cannot be found in the name scope" right after the token arrived → the app died | No test opened a real window for `Loaded` to fire | Put the animation on the element itself, no `TargetName`; tests open a real window |
| No global error boundary → one rendering error killed the whole process | Missing entirely | `DispatcherUnhandledException` + `UnobservedTaskException`: write a redacted report, ask Continue/Quit |
| Valid token on disk, yet reopening still demanded Sign in | No real token had ever been on disk during tests | The first flow reads the restored session state; reload the scopes |
| A background-thread callback mutated a bound `ObservableCollection` → the table went empty with no error | Unit tests have no CollectionView | The UI-side adapter captures the `SynchronizationContext` at construction and `Post`s every callback back to it; drop `ConfigureAwait(false)` in UI code |

How to run for real when the owner is away: drive the app with UI Automation (`System.Windows.Automation` from PowerShell: find by Name + ControlType, `SelectionItemPattern`/`InvokePattern`/`ValuePattern`/`TogglePattern`), capture the window with `Graphics.CopyFromScreen`, read the `Application` Event Log when the app exits unexpectedly. Only the **Authorize step in the browser** needs a real person.

---

## 11. Checklist to carry into a new project

- [ ] Create the OAuth client on the developer's own account; Client URL a real domain if you plan to go Public; redirect `http://localhost:<fixed port>/callback`; tick the Required scopes.
- [ ] Get the scope **identifiers** from the popup or `/oauth/scopes`; write the label ↔ identifier table down in one place; tests pin the identifier.
- [ ] Run a 100-line prototype through the whole loop before writing product code: authorize → callback → token exchange → `GET /accounts`. Record the real responses (redacted).
- [ ] Public client + PKCE S256 + `state`; **no** secret; `scope` required; do not ask for `offline_access`.
- [ ] Listener: fixed port, two separate loopback listeners, one request, 5 minutes, self-contained callback page; dedicated error codes for "port taken" and "browser will not open".
- [ ] Token exchange: no retry; check `token_type` is `bearer`, and that `access_token` and `scope` are present; default to 3600 if `expires_in` is missing.
- [ ] Store with DPAPI including `expires_at` and the scopes; restore when the app opens; **never** open a browser by itself; sign in on demand, stitched into the operation.
- [ ] Preflight with `GET /accounts`; read 9999/1010/10000 with their real meanings; compare the account against last time.
- [ ] Whatever has to keep running uses its own credential and does not depend on the access token.
- [ ] Global error boundary in place before the first real run.
- [ ] Run for real as soon as the wiring is done; every bug that surfaces gets a regression test **at the right level** (real window, real binary), not just a unit test.
- [ ] Before going Public: real-domain Client URL, logo, verification TXT; check whether consent still reappears.

---

## 12. Still open

- Revoke (`/oauth2/revoke`) has never been called for real.
- Whether a Public + Verified client skips consent.
- Whether `POST /oauth/clients` accepts `offline_access`.
- Behaviour when a user unticks an **optional** scope (the test client has Required scopes only).
- The Device Authorization Grant is available but untried; worth considering if the default browser is often signed in to the wrong account.
