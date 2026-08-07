---
parent: uauth-service
prefix: SVC-SESS
---

# sessions

## Context and Design Philosophy

sessions owns access/refresh token issuance, the session record, rotation,
revocation, and device management. A session is created whenever
login-methods successfully verifies proof of identity for any method; it is
consulted and re-issued on refresh; it is deleted on logout; and it backs the
"your active devices" view.

## Access tokens vs. refresh tokens

Access tokens and refresh tokens play different roles and are shaped
differently:

- **Access tokens** are short-lived, self-contained JWTs, signed with a
  KMS-managed key. They are proof that a session was valid recently, not the
  session itself — their short TTL bounds exposure if one leaks, and their
  self-contained signature lets them be verified without a database lookup
  for the common case.
- **Refresh tokens are the session.** Each successful login, regardless of
  method, creates one session — one refresh token tied to one device/client
  instance — recorded server-side in a sessions table (DynamoDB). Refresh
  tokens are opaque random strings, not JWTs (see Decisions & Alternatives).

## Session record

A session record holds: session ID, user ID, a hash of the current refresh
token plus a hash of the immediately-previous one with its supersession
timestamp (never the raw token values, same principle as never storing a
plaintext password — § Rotation concurrency and recovery), device/client
metadata (for a "your active devices" view), created-at/last-used
timestamps, a fixed expiration timestamp (§ Retention), and a revoked flag.

## Rotation and revocation

Refresh tokens rotate on every use: each refresh issues a new refresh token
and invalidates the one just used. If an already-rotated-out refresh token is
ever presented again, that's a signal of token theft, and the entire session
is revoked.

The sessions table is only touched at login, refresh, logout, and when
listing active devices — not on every regular request, which is what keeps
per-request access-token verification cheap regardless of whether that
verification happens locally or via network call (uauth-service § Key
Design Decisions).

A burst of failed refresh attempts against one session is a theft signal
worth monitoring, since valid refresh tokens are high-entropy and not
realistically guessable (uauth-service § Abuse protection).

## Rotation concurrency and recovery

Two near-simultaneous refresh calls presenting the same token (e.g. a client
retry racing the original request) are resolved by the conditional write
(§ Sequences → Refresh): only one of the two can win and actually rotate.

The loser isn't treated as theft. Both calls converge on holding the
identical current token — the client just stores whatever refresh token it's
given, so this needs no special handling client-side. This convergence is
what keeps the race from forking session state into two valid tokens, and
it's why the session record keeps one prior generation (§ Session record)
rather than just the current token.

A token presented outside the grace window, or more than one generation
behind, doesn't get this treatment — a legitimate single owner would never
present a token that stale, so it's treated as genuine reuse of a token
that's supposed to be dead, and the session is revoked (§ Sequences →
Refresh). The grace window's duration is a tuning parameter, not yet chosen
(§ Open Questions).

If the client never receives the response carrying a freshly-rotated refresh
token (a crash or dropped connection after the server-side write), no
special recovery path exists: the client is left holding a dead refresh
token, and the next refresh attempt fails. This is an acceptable outcome —
the client falls back to a full login, the same behavior as any other
refresh failure.

## Retention

Each session has a fixed expiration timestamp set once at login — a long,
flat TTL that rotation never resets, regardless of how often the session is
used. An actively-used session still eventually requires a full re-login
once the fixed window elapses; an abandoned session expires on the same
schedule. This keeps expiration independent of any definition of "activity,"
and applies uniformly across every login method rather than depending on any
one method's friction (§ Decisions & Alternatives).

Revoked and expired session records are removed via a DynamoDB TTL
attribute, not an active cleanup job — the record is written with an
expiration timestamp (this fixed session lifetime, or, for a revoked
session, revocation time plus a short retention window for audit purposes)
and DynamoDB garbage-collects it natively.

## Sequences

Ordered steps only — rationale in § Rotation concurrency and recovery and
§ Retention.

### Refresh

1. Client sends `POST /refresh` with its current refresh token.
2. sessions looks up the session by the token's hash.
   - If the token matches the session's *current* hash, sessions rotates:
     it generates a new refresh token and conditionally writes it as the
     current hash — only if the stored hash still matches the presented
     token — moving the old current hash to previous, with a supersession
     timestamp, and mints a new access token.
   - If the token matches the session's *immediately-previous* hash and is
     within the grace window, sessions returns the session's current access
     and refresh tokens without rotating again.
   - Otherwise (no match, or a previous-hash match outside the grace
     window), sessions revokes the session and returns
     `{ error: "invalid_session" }`.
3. On either successful branch, sessions returns
   `{ accessToken, refreshToken }` — the client stores whatever it's given,
   without needing to distinguish a freshly-rotated token from a
   grace-window one.

### Logout

1. Client sends `POST /logout` with its current refresh token.
2. sessions revokes the session identified by the token's hash.
3. sessions returns `{}`. The client clears local state regardless of
   whether the call succeeds.

## Interface

| Endpoint | Method | Auth | Request | Response |
|---|---|---|---|---|
| `/refresh` | POST | none (the refresh token itself is the credential) | `{ refreshToken: string }` | Success: `{ accessToken: string, refreshToken: string }` — the returned `refreshToken` is either newly-rotated or, in the grace-window case, the session's already-current one (§ Rotation concurrency and recovery); the client always just stores whatever it's given. Failure: `{ error: "invalid_session" }` if the token is unknown, expired, or reused outside the grace window. |
| `/logout` | POST | none (the refresh token itself is the credential) | `{ refreshToken: string }` | `{}` — revokes the session identified by the refresh token. Best-effort from the client's perspective: local state is cleared regardless of whether this call succeeds. |

The access token returned by `/refresh` is a JWT whose payload includes at
least `sub` (the user ID) and standard `exp`/`iat` claims; whether it also
carries profile fields is pending the full-profile-vs-identity-only question
below.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Refresh token format | Opaque random string, looked up against the sessions table | Self-contained JWT refresh token | A JWT refresh token can't be revoked before it naturally expires without an explicit revocation check on every use, which erases the "no DB lookup" benefit that motivates using a JWT in the first place. Refresh tokens live far longer than access tokens, so an unrevocable window is a much larger risk here than for access tokens. An opaque token forces a server-side lookup on every refresh, which is acceptable because refresh happens rarely (roughly once per access-token TTL) rather than on every request — and that lookup is also the natural place to add rotation and reuse detection, giving instant, reliable revocation exactly where it matters most. |
| Concurrent-refresh handling | Conditional write + short grace window accepting the immediately-previous token | Treat any reuse of a rotated-out token as theft, no grace window | A hard "any reuse is theft" rule produces false positives from ordinary retries and multi-tab clients, which are expected, benign traffic, not attacks — a short grace window absorbs those while still catching reuse far outside any plausible retry timing. |
| Session record cleanup | DynamoDB TTL attribute (native expiry) | An active/scheduled cleanup job | Native TTL requires no additional infrastructure or scheduled compute, consistent with the project's cheap-ops goal. |
| Session expiration model | Long, fixed TTL set at login, not extended by activity or rotation | Sliding TTL, reset on each rotation | Sliding TTL's cost — defining "activity," resetting expiration on every rotation, reasoning about when a session actually dies — doesn't depend on login method, but its benefit (never bothering an active user) does: it matters most when re-login is painful (passwords, MFA) and least when it's cheap (Google's native picker). Since this model is meant to apply uniformly across every login method, not just the low-friction ones available today, justifying it by any one method's friction doesn't hold up as new methods are added. A sufficiently *long* fixed TTL captures most of the same practical benefit instead — an active user rarely hits the cap regardless of method — without the added complexity, and a fixed maximum session lifetime is a defensible security practice on its own terms: it bounds how long any single authentication event, compromised or not, stays valid without a fresh proof of identity. |

## Open Questions & Future Decisions

### Deferred

1. Access-token TTL, refresh-token TTL, and the rotation grace-window
   duration are not yet chosen.
2. Whether server-side `getCurrentUser` (see server-sdk) needs full profile
   data on every check or only a verified identity (user ID + claims) is not
   yet resolved — affects whether verification needs a DB read on every call
   or can stay signature-only for the common case.
3. Session-listing/device-management API shape (what a client actually calls
   to see/revoke other sessions) is not yet designed.
4. Whether the theft-revocation path (reuse outside the grace window) should
   also revoke every *other* session on the account, not just the one
   affected, is not yet decided.

## References

- Parent: `docs/intent/uauth-service/uauth-service-design.md`
- Root HLD: `docs/high-level-design.md`
