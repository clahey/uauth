---
parent: client-sdk
prefix: CLIENT-WEB
---

# client-sdk / web

## Context and Design Philosophy

The web client SDK implements the client-sdk contract in TypeScript for the
JS/TS web frontend. It is the first client SDK to reach MVP, alongside
android and server-sdk.

## Google login ceremony

`loginWithGoogle()` uses the standard OIDC redirect flow: the browser is sent
to Google's authorization endpoint, the user authenticates on Google's own
domain, and Google redirects back with an authorization code that gets
exchanged for a Google ID token. The resulting ID token is sent to
uauth-service/login-methods for verification. Unlike Android, there is no
native-SDK path to bypass this redirect — see root HLD § Options Under
Consideration for why this doesn't change the overall comparative effort
read.

## Token storage

The refresh token (the session — see uauth-service/sessions) must be stored
between page loads. Not yet decided: browser `localStorage` vs. an httpOnly
cookie — these have materially different XSS exposure and the choice hasn't
been made yet. See Open Questions.

## CSRF protection and redirect errors

Before redirecting to Google, `loginWithGoogle()` generates a cryptographically
random `state` value and stores it (e.g. `sessionStorage`) alongside the
pending login attempt. On the callback, the returned `state` must match the
stored value before the authorization code is exchanged; a missing or
mismatched `state` is rejected as a failed login rather than proceeding —
this is what prevents an attacker from tricking a user's browser into
completing a login the user didn't initiate.

If Google's redirect carries an `error` query parameter instead of an
authorization code (the user denies consent, or a provider-side failure),
this is treated as an ordinary failed login — the same "not logged in"
outcome as any other login failure — not a crash.

## Multi-tab behavior

Two tabs racing a `getToken()`-driven refresh on the same stored refresh
token is the client-visible version of the rotation race handled server-side
by sessions' grace window (see uauth-service/sessions § Rotation concurrency
and recovery) — no tab-coordination logic is needed for correctness, since
the server already absorbs this race safely. Coordinating tabs to avoid the
redundant refresh call in the first place (e.g. via `BroadcastChannel`) is a
possible future efficiency improvement, not a correctness requirement — see
Open Questions.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| CSRF protection on the OIDC redirect | Random `state` value, generated and checked client-side | No `state` check | Without `state`, the auth-code exchange has no way to confirm the callback corresponds to a login this browser actually initiated — a standard, well-established OIDC/OAuth2 requirement, not specific to this project. |
| Multi-tab refresh races | Rely on sessions' server-side grace window; no client-side tab coordination | Cross-tab locking/coordination (e.g. `BroadcastChannel`) to prevent redundant refresh calls | The server-side fix already makes concurrent refreshes safe; client-side coordination would only reduce redundant calls, which is an efficiency concern, not a correctness one — not worth the added complexity now. |

## Open Questions & Future Decisions

### Deferred

1. Refresh token storage location (`localStorage` vs. httpOnly cookie) is
   not yet decided — a real security-relevant choice given Tenet #1
   (security first).
2. `getToken()`'s transparent-refresh implementation (when to proactively
   refresh vs. refresh on 401, concurrency handling if multiple requests
   need a token at once) is not yet designed.
3. Cross-tab coordination (e.g. `BroadcastChannel`) to avoid redundant
   refresh calls across open tabs is a possible future efficiency
   improvement, not yet designed.

## References

- Parent: `docs/intent/client-sdk/client-sdk-design.md`
- Root HLD: `docs/high-level-design.md`
