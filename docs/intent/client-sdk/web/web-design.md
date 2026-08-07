---
parent: client-sdk
prefix: CLIENT-WEB
---

# client-sdk / web

## Context and Design Philosophy

The web client SDK implements the client-sdk contract in TypeScript for the
JS/TS web frontend. It reaches MVP alongside android and server-sdk.

## SDK configuration

The web SDK must be configured with a `googleWebClientId` — the consuming
project's own Google OAuth Web client ID, registered with uauth-service
ahead of time (uauth-service/login-methods § Google client registration) —
before `loginWithGoogle()` can be called. This isn't a secret; it's how
uauth-service knows both which origin is allowed to receive that project's
login results and which Google credentials (and consent-screen branding) to
use for the exchange. Exact configuration mechanism (a build-time constant
vs. a runtime `configure()`-style call) is not yet decided (§ Open
Questions).

## Google login ceremony

`loginWithGoogle()` hands the entire Google exchange to uauth-service rather
than performing any part of it itself (uauth-service/login-methods § Login
sequences → Google (Web)). Web never talks to Google directly and never
handles a Google authorization code itself; unlike Android, there's no way
to obtain a Google ID token client-side, so the whole exchange has to happen
somewhere else (root HLD § Options Under Consideration).

Because the exchange happens in a popup rather than a full-page redirect,
the main page never navigates away — `loginWithGoogle()` resolves in place
once the popup's result arrives, like any other async call, and the calling
app decides what to do next.

## Token storage

The refresh token (the session — uauth-service/sessions) arrives at the main
page via `postMessage` (§ Google login ceremony) and must be stored between
page loads. Since delivery happens through JavaScript running on the web
app's own origin, `localStorage` is the storage mechanism that fits without
adding another moving part — an httpOnly cookie can only be set by a server
response, and no server in this flow is positioned to set one scoped usefully
to the web app. This follows from the popup + `postMessage` design rather
than being an independent choice (§ Decisions & Alternatives).

## postMessage integrity and failure handling

Two checks protect the result delivered via `postMessage`:

- **Origin check** — the main page verifies `event.origin` matches
  uauth-service's own origin before trusting anything in the message, so an
  unrelated page can't forge a "login succeeded" message.
- **Nonce check** — the client generates an opaque `clientNonce` before
  opening the popup (uauth-service/login-methods § Interface) and verifies
  it's echoed back in the message payload, confirming the result
  corresponds to the popup this specific call opened, not a stray or
  replayed message.

If Google's own redirect to uauth's callback carries an `error` parameter
instead of a code (the user denies consent, or a provider-side failure), or
uauth's own `state` check fails, the popup's final page posts a failure
result instead of tokens — from `loginWithGoogle()`'s caller's perspective,
this is the same "not logged in" outcome as any other login failure, not a
crash.

## Multi-tab behavior

Two tabs racing a `getToken()`-driven refresh on the same stored refresh
token is the client-visible version of the rotation race handled server-side
by sessions' grace window (uauth-service/sessions § Rotation concurrency and
recovery) — no tab-coordination logic is needed for correctness, since the
server already absorbs this race safely. Coordinating tabs to avoid the
redundant refresh call in the first place (e.g. via `BroadcastChannel`) is a
possible future efficiency improvement, not a correctness requirement (§ Open
Questions).

## Interface

TypeScript signatures implementing the client-sdk contract:

```typescript
async function loginWithGoogle(): Promise<LoginResult>;
async function logout(): Promise<void>;
function getCurrentUser(): CurrentUser | null;
async function getToken(): Promise<string | null>;

type LoginResult =
  | { ok: true; user: CurrentUser }
  | { ok: false };

interface CurrentUser {
  userId: string;
  name?: string;
  picture?: string;
  email?: string;
}
```

`{ ok: false }` covers a rejected/failed popup result (Google's own error, a
state/nonce mismatch, or a server-side verification failure — § postMessage
integrity and failure handling) — all collapse to the same "not logged in"
outcome from the caller's perspective.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Google login delivery mechanism | Popup window + `postMessage`, with uauth-service hosting the entire OAuth redirect dance | (1) Full-page redirect with tokens delivered in the URL fragment; (2) full-page redirect with a one-time-use exchange code; (3) full-page redirect with the refresh token set as a cross-site httpOnly cookie on uauth-service's own domain | All three redirect-based alternatives require reconstructing "where the user was" and reloading the page. (3) additionally depends on `SameSite=None` cookies being sent on cross-site requests from the web app's origin to uauth-service's — exactly the category of behavior browser vendors (Safari ITP, Chrome's third-party-cookie deprecation) are actively restricting, a durability risk for a multi-year project. Popup + `postMessage` avoids cookies and cross-site storage access entirely, needs no page navigation to reconstruct, and is a well-established production pattern — confirmed for both Firebase Auth's `signInWithPopup` and Auth0's `loginWithPopup`, which uses `postMessage` the same way. Both ecosystems document real gotchas worth carrying forward: `Cross-Origin-Opener-Policy`/`Cross-Origin-Embedder-Policy` headers (set for unrelated reasons, e.g. `SharedArrayBuffer` access) can break the `postMessage` origin match, and popup-based sign-in is documented as unreliable on mobile browsers and in standalone/PWA mode, where Firebase's own guidance recommends redirect instead (§ Open Questions). |
| Integrity protection on the login result | Origin check + client-generated nonce, verified on the `postMessage` payload | Trusting any `postMessage` claiming to be from uauth-service without verification | Without an origin check, any page could forge a "login succeeded" message; without a nonce tied to the specific popup opened, a stray or replayed message could be misread as this call's result. Google's own `state` parameter is owned and validated by uauth-service server-side (uauth-service/login-methods § Interface), since web never talks to Google directly. This is the receiving side's check; the complementary sending-side protection — uauth-service only ever targeting a *registered* origin, never a caller-supplied one — is the `googleWebClientId` registry (uauth-service/login-methods § Google client registration), without which any site could receive another user's session regardless of what this page checks. |
| Multi-tab refresh races | Rely on sessions' server-side grace window; no client-side tab coordination | Cross-tab locking/coordination (e.g. `BroadcastChannel`) to prevent redundant refresh calls | The server-side fix already makes concurrent refreshes safe; client-side coordination would only reduce redundant calls, which is an efficiency concern, not a correctness one — not worth the added complexity now. |

## Open Questions & Future Decisions

### Deferred

1. `getToken()`'s transparent-refresh implementation (when to proactively
   refresh vs. refresh on 401, concurrency handling if multiple requests
   need a token at once) is not yet designed.
2. Cross-tab coordination (e.g. `BroadcastChannel`) to avoid redundant
   refresh calls across open tabs is a possible future efficiency
   improvement, not yet designed.
3. Popup-blocker handling (`window.open` must be called synchronously
   within the triggering click handler) is not yet addressed.
4. Mobile-browser and standalone/PWA-mode popup reliability: both Firebase
   Auth and Auth0's popup flows are documented as unreliable in these
   contexts, with redirect recommended instead — whether uauth needs a
   redirect-based fallback path for these cases, or accepts the limitation,
   is not yet decided. If a fallback is built, Auth0's redirect model is the
   better template, not Firebase's: Auth0 stores a PKCE-style transaction in
   the app's own `sessionStorage` before redirecting (same-origin, set
   before navigating away), lands the callback on the *consuming app's own*
   registered callback URL, and has the app's own JS complete the exchange
   via a direct CORS fetch — no cookie involved anywhere. Firebase's
   equivalent requires hosting Firebase-provided helper files under the
   consuming app's own domain plus a hidden iframe bridge, which would
   impose real integration burden on every consuming project and sits
   awkwardly against the minimal-integration-surface goal. Following
   Auth0's model would mean uauth needs its own concept of a per-project
   registered callback URL — a new, uauth-internal registration, distinct
   from (and not a change to) the single shared Google redirect URI uauth
   already owns.
5. `Cross-Origin-Opener-Policy`/`Cross-Origin-Embedder-Policy` headers (if
   ever set on the web app or uauth-service's callback page for unrelated
   reasons) can break `postMessage` origin matching — not yet addressed,
   since neither side sets these today.
6. How the SDK is configured with its `googleWebClientId` (a build-time
   constant vs. a runtime `configure()`-style call) is not yet decided
   (§ SDK configuration).

## References

- Parent: `docs/intent/client-sdk/client-sdk-design.md`
- Root HLD: `docs/high-level-design.md`
