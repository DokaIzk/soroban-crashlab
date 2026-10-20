# GitHub OAuth callback flow (#1559)

## Sequence diagram

```mermaid
sequenceDiagram
    autonumber
    participant U as User (browser)
    participant L as Login route<br/>/api/auth/github/login
    participant G as GitHub
    participant C as Callback route<br/>/api/auth/github/callback
    participant S as Session cookie<br/>(crashlab_github_session, HttpOnly)

    U->>L: GET /api/auth/github/login?next=/runs/1
    L->>L: issueState() -> HMAC-signed token<br/>(nonce.hmac-sha256)
    L->>L: Set-Cookie github_oauth_state<br/>(HttpOnly, Secure*, SameSite=Lax, maxAge=600s)
    L->>G: 302 authorize?client_id&redirect_uri&state=<token>&scope
    G->>U: user grants access
    G->>C: 302 callback?code=...&state=<token>[&next=/runs/1]
    C->>C: verifyState(stored cookie)<br/>rejects tampered / unsigned state
    C->>C: constantTimeEqual(query state, cookie state)<br/>rejects mismatch
    alt state missing / mismatch / invalid
        C->>U: 400 + clear state + next cookies
    else valid
        C->>C: exchange code for token (stub; code never logged)
        C->>S: Set-Cookie session<br/>(HttpOnly, short configurable TTL)
        C->>U: 302 <safeNext> + clear state + next cookies
    end
```

* `Secure` only when `NODE_ENV=production`.

## Security properties (#1559)

| # | Property | Requirement | How it is met |
|---|----------|-------------|---------------|
| (a) State binding | Cryptographically bound, HttpOnly, Secure, SameSite=Lax, 10-min expiry | `apps/web/src/lib/github-oauth-state.ts` mints `<nonce>.<hmac-sha256>`; cookie uses `httpOnly`, `secure` (production), `sameSite: 'lax'`, `maxAge: 600` |
| (b) Redirect control | Never redirect to a caller-influenced external URL | `safeRedirectPath` with a strict allowlist of internal path prefixes (`/`, `/runs`, `/dashboard`, `/settings`, `/integrations`, `/alerts`); everything else falls back to `/` |
| (c) Token storage | Exchange token server-side, short TTL, never serialized to client | Authorization code is never logged; the session is an HttpOnly signed cookie (unreadable by client JS); TTL via `CRASHLAB_GITHUB_SESSION_TTL_SECONDS` |
| (d) Replay / raced callbacks | Single-use state | State + next cookies are cleared on every outcome; a replayed callback finds no cookie and is rejected with 400 |

## Configuration

- `NEXT_PUBLIC_GITHUB_CLIENT_ID` — GitHub OAuth client ID (required; 503 otherwise).
- `CRASHLAB_GITHUB_STATE_SECRET` — state signing secret. Falls back to `CRASHLAB_GITHUB_SESSION_SECRET`. Set in production; without it, state is an unbound random nonce and a warning is logged.
- `CRASHLAB_GITHUB_SESSION_TTL_SECONDS` — session cookie TTL (default 7 days; set short in production).
- `NODE_ENV=production` — enables the `Secure` cookie attribute.

## Files

- `apps/web/src/app/api/auth/github/login/route.ts`
- `apps/web/src/app/api/auth/github/callback/route.ts`
- `apps/web/src/lib/github-oauth-state.ts`
- `apps/web/src/app/api/auth/github/callback/route.test.ts`
