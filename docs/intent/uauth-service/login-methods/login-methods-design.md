---
parent: uauth-service
prefix: SVC-LOGIN
---

# login-methods

## Context and Design Philosophy

login-methods verifies proof of identity for each supported login method and,
on success, hands off to sessions for token issuance. Each method has its own
client-side ceremony (see client-sdk for the client half of each flow), but
all of them converge on the same backend contract: the client obtains proof
of identity for that method, hands it to uauth in one call, and login-methods
verifies it and requests a new session from sessions.

The set of supported login methods is closed and owned by uauth itself — not
something a consuming project supplies its own implementation of or extends.
Verification of each method's proof is uauth's own internal strategy per
method (JWKS check, password hash check, OTP check, WebAuthn verification);
this is an implementation detail, not exposed integration surface. See
client-sdk § Key Design Decisions for the client-facing consequence of this.

## Supported methods

### Google (OIDC) — implemented for MVP

The two client platforms reach login-methods differently, since only Android
can obtain a Google ID token client-side (§ Login sequences; client-sdk/web
§ Google login ceremony; client-sdk/android § Google login ceremony).

Once login-methods holds a verified Google ID token (either sequence's final
verification step), it looks up or creates the corresponding account (see
accounts) keyed by Google's `sub` claim, and passes basic profile claims
(`name`, `picture`, `email`) through for display — no custom-settable
profile fields yet (see accounts).

### Email/password — future

No external ceremony. The client sends credentials directly to uauth in one
call; uauth checks the stored password hash. Not yet designed in detail (no
password storage, hashing library choice, or account-creation flow
specified) — see Open Questions.

### OTP — future

Two API round trips, no external ceremony: the client requests a code
(delivered via SES/SNS), then submits the code back for verification. Not
yet designed in detail — see Open Questions.

### Passkeys (WebAuthn) — future

uauth issues a random challenge, the client passes it to the platform's
native WebAuthn API (not a third-party SDK), and returns the resulting
signed assertion to uauth for verification. Not yet designed in detail (no
challenge storage/expiry, credential storage, or relying-party configuration
specified) — see Open Questions.

## Login sequences

Ordered steps only — rationale in § Supported methods and § Google client
registration.

### Google (Android)

1. Client calls Android's Credential Manager, requesting a Google ID token
   with the app's registered `googleWebClientId` passed as `serverClientId`.
2. Credential Manager presents a native account picker and returns a
   Google-signed ID token, or no credential if the user has no Google
   account on the device or cancels the picker.
3. Client sends the ID token to `POST /login/google`.
4. login-methods verifies the token's signature against Google's public
   JWKS, checks `iss`/`exp`, and checks `aud` against the registered set of
   `googleWebClientId`s.
5. On success, login-methods proceeds to account lookup-or-create and
   session issuance (see Verification flow), returning
   `{ accessToken, refreshToken, user }`. On failure, it returns
   `{ error: "authentication_failed" }`.

### Google (Web)

1. Client opens a popup pointed at `GET /login/google/start`, passing its
   configured `googleWebClientId` and a client-generated `clientNonce`.
2. login-methods checks `googleWebClientId` against the registry. If
   unregistered, it rejects immediately, without redirecting to Google.
3. If registered, login-methods redirects the popup to Google's
   authorization endpoint, using the registered project's own OAuth
   credentials, uauth-service's own callback as `redirect_uri`, and a
   uauth-generated `state`.
4. The user completes or denies consent with Google, which redirects the
   popup to `GET /login/google/callback` with `?code=...&state=...` or
   `?error=...`.
5. login-methods validates `state` and, if present, exchanges `code` for an
   ID token server-side using the registered project's `googleClientSecret`.
6. login-methods verifies the resulting ID token as in Google (Android)
   step 4, then proceeds to account lookup-or-create and session issuance
   (see Verification flow). Google's own `?error=...`, a `state` mismatch,
   or a failure at any step up to here all skip straight to step 7 as a
   failure.
7. The callback endpoint returns an HTML page whose script posts the result
   to `window.opener` via `postMessage`, targeted at the origin registered
   for this `googleWebClientId` (never `'*'`, never a caller-supplied
   value), and closes the popup. Success:
   `{ ok: true, clientNonce, accessToken, refreshToken, user }`. Failure:
   `{ ok: false, clientNonce }`.
8. The main page's `loginWithGoogle()` call, listening for the message,
   verifies `event.origin` and the echoed `clientNonce` (see client-sdk/web
   § postMessage integrity and failure handling) before resolving.

## Verification flow

Every method's sequence converges on the same final step once proof of
identity succeeds: login-methods calls accounts to look up or create the
corresponding account, then calls sessions to issue a new session. Account
lookup-or-create is idempotent (safe to repeat with the same
identity-provider mapping — see accounts), so if session issuance fails after
account creation succeeded, the client simply retries the whole login call:
it will find the already-created account and proceed normally. No rollback
of account creation is needed.

No deduplication is applied to repeated submissions of the same still-valid
proof (e.g. a client retry after a dropped response): each successful
verification issues a new session. This is safe and consistent with a user
legitimately logging in from multiple devices or tabs — an extra session from
a retry is a normal outcome, not an error condition.

A failed verification (bad signature, expired token, wrong `aud`/`iss`, or
any other check failure) returns a single generic failure reason to the
client (e.g. `authentication_failed`) rather than which specific check
failed — the client's only actionable response is to re-attempt the method's
ceremony, and a detailed reason would help an attacker tune a malicious
token.

## Google client registration

Two separate problems, addressed by the same registry.

**postMessage delivery (web only).** The web login ceremony (see
client-sdk/web § Google login ceremony) hands a freshly-issued session to
whichever window receives uauth's `postMessage` — and without knowing in
advance which origin that should be, uauth has no safe way to restrict
delivery to the legitimate caller. Any site could open the same popup, and
if uauth doesn't know which origin is legitimate, it can only choose between
broadcasting the result to anyone listening (`'*'` as `postMessage`'s
target, unsafe) or trusting an origin value the caller itself supplies
(equally unsafe, since a malicious caller would just supply whichever origin
lets it receive the message).

**Consent-screen branding (both platforms).** Google's OAuth consent
screen — the app name and logo shown to the user during login — is
configured per Google Cloud *project*, shared across every OAuth client
within that project. If uauth-service used one shared Google Cloud project
for every consuming project, every login — web or Android — would show
uauth's own name to the user instead of the consuming project's, regardless
of which app the user thinks they're logging into. This affects Android too:
the `serverClientId` Android passes to Credential Manager to obtain an ID
token is a Web-type client ID within a Google Cloud project, and that
project's consent-screen configuration is what the user sees, the same
mechanism as web.

Both problems are fixed by requiring each consuming project to own its
*own* Google Cloud project — its own consent-screen branding, its own
`googleWebClientId` and `googleClientSecret` — registered with uauth ahead
of time, set up out-of-band (as part of uauth's own deployment, not
self-service — see uauth-service § Open Questions for infrastructure/
deployment work still undesigned). This is the same pattern Auth0 uses:
generic shared "dev keys" for prototyping, but production apps supply their
own Google credentials so users see the real app's identity, not Auth0's.

For **web**, the registry lookup happens at `/login/google/start`, before any
redirect to Google (§ Login sequences → Google (Web), steps 1–3) — the
resolved, registered origin and the project's `googleClientSecret` carry
through server-side to the callback (steps 5–7) rather than being
reconstructed from the request each time. `googleWebClientId` itself isn't a
secret: knowing or guessing a real one doesn't help an attacker, since uauth
always uses the *registered* origin for that ID as `postMessage`'s target,
never a value supplied by whoever opened the popup — an attacker's page
simply never receives the message, because the browser only delivers a
targeted `postMessage` to a window actually at the matching origin.

For **android**, no extra request parameter is needed: the ID token Android
sends to `/login/google` already carries the project's `googleWebClientId`
in its `aud` claim (Android declared it as `serverClientId` when requesting
the token). login-methods' `aud` check (§ Login sequences → Google (Android),
step 4) is a registry lookup — `aud` must match a *registered*
project's `googleWebClientId`, not one single uauth-wide expected value.

## Interface

Each supported method has its own login endpoint(s), part of uauth-service's
public surface (see uauth-service § API surfaces). Only Google is defined
for MVP; the others get their own endpoint once the corresponding method is
designed (see Open Questions).

Google has two different entry points, since android and web obtain proof
of identity differently (see Supported methods above):

| Endpoint | Method | Auth | Request | Response |
|---|---|---|---|---|
| `/login/google` | POST | none | `{ idToken: string }` | Success: `{ accessToken: string, refreshToken: string, user: { userId: string, name?: string, picture?: string, email?: string } }`. Failure: `{ error: "authentication_failed" }` (§ Verification flow; also covers an unregistered `aud`). Used by android, which obtains a Google ID token directly via Credential Manager. |
| `/login/google/start` | GET | none | Query params: `googleWebClientId` (§ Google client registration), `clientNonce` (opaque, generated by the web SDK before opening the popup) | Redirects to Google's authorization endpoint, using the registered project's own Google OAuth credentials, with uauth-service's own callback registered as the `redirect_uri` and a uauth-generated, uauth-validated `state`. Rejects immediately (no redirect to Google) if `googleWebClientId` isn't registered. `clientNonce` is carried through unchanged to the final `postMessage` payload from `/login/google/callback`, so the opener can confirm the result corresponds to the popup it actually opened. Used by web (client-sdk/web § Google login ceremony). |
| `/login/google/callback` | GET | none | Google's redirect: `?code=...&state=...` (or `?error=...`) | An HTML response, not JSON — the code is exchanged server-side using the registered project's `googleClientSecret`, then the response's script posts the result to `window.opener` via `postMessage`, targeted at the origin registered for this attempt's `googleWebClientId` (§ Google client registration) — never `'*'` and never a caller-supplied value. Success payload: `{ ok: true, clientNonce, accessToken, refreshToken, user }`. Failure payload (bad code, state mismatch, unregistered client, or Google's own `error`): `{ ok: false, clientNonce }` (client-sdk/web § postMessage integrity and failure handling). |

`user` in a success response is the pass-through profile data described in
client-sdk — captured once at login from the method's own claims, not
re-fetched on later calls. `accessToken` and `refreshToken` are as described
in uauth-service/sessions.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Ownership of the supported-method set | Closed set, owned by uauth itself | Allow consuming projects to register/extend custom login methods | Matches the *Minimal, stable identity surface* Tenet — no bespoke integration APIs. A per-project-extensible method set would mean uauth verifying arbitrary consuming-project-supplied logic, exactly the kind of bespoke surface the project avoids. |
| Restricting who receives the web login's `postMessage` result | A uauth-managed registry: each web consuming project registers a `googleWebClientId` mapped to its allowed origin(s); `postMessage`'s target is always the *registered* origin, never a caller-supplied value | (1) Broadcast via `postMessage(payload, '*')`; (2) trust an `origin` parameter supplied by the caller opening the popup | (1) lets any page that opens the popup receive the result — since the popup's content (a real Google login, a real uauth domain) looks completely legitimate to the user, any site could harvest working sessions just by embedding a "Sign in with Google" button. (2) doesn't fix this: a malicious caller would simply supply whichever origin it wants the message delivered to, since uauth has no way to verify a self-reported value. Only a registry uauth itself controls closes the gap — the registered ID isn't secret, since the security comes from uauth choosing the target origin, not from the ID being hidden. |
| Google Cloud project ownership per consuming project | Each consuming project owns its own Google Cloud project, consent-screen branding, and OAuth credentials, registered with uauth | One shared Google Cloud project for uauth-service itself, used across every consuming project | Consent-screen branding (app name, logo) is configured per Google Cloud project, not per OAuth client within it — a shared project means every consuming project's users see "uauth" during login instead of the app they actually think they're using, for both web and Android. This is the same problem Auth0 solves the same way: generic shared credentials for prototyping, real per-app credentials for anything user-facing. |

## Open Questions & Future Decisions

### Deferred

1. Email/password: password storage/hashing library, account-creation flow,
   and brute-force protection are not yet designed (see uauth-service §
   Abuse protection for the general requirement).
2. OTP: delivery mechanism configuration (SES vs. SNS), code TTL, and rate
   limiting are not yet designed.
3. Passkeys: challenge storage/expiry, credential storage, and relying-party
   configuration are not yet designed.
4. Exact JWKS-caching strategy for Google's public keys (TTL, refresh-on-
   miss behavior) is not yet designed.
5. Whether the generic failure reason should be differentiated further for
   legitimate client UX needs (e.g. "your device clock is wrong" vs. "try
   again") without leaking security-relevant detail is not yet resolved.
6. The registration process itself — how a consuming project actually gets
   a `googleWebClientId`/`googleClientSecret`/allowed-origin entry added
   (presumably as part of uauth's own IaC deployment, tied to
   uauth-service's undesigned infrastructure question), and who's
   responsible for creating that project's own Google Cloud project and
   OAuth consent screen in the first place — is not yet designed.
7. Storage shape is not yet designed: the client secret is sensitive enough
   to likely warrant AWS Secrets Manager rather than sitting in the same
   DynamoDB row as the (non-secret) client ID and allowed origin, but this
   isn't decided.
8. The exact response uauth returns when `/login/google/start` is called
   with an unregistered `googleWebClientId`, or `/login/google` is called
   with an unrecognized `aud`, (an HTML error page for the former, since
   that endpoint is hit via browser navigation, not a JS fetch) is not yet
   specified.

## References

- Parent: `docs/intent/uauth-service/uauth-service-design.md`
- Root HLD: `docs/high-level-design.md`
