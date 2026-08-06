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

A session record holds: session ID, user ID, a hash of the refresh token
(never the raw value, same principle as never storing a plaintext password),
device/client metadata (for a "your active devices" view), created-at/
last-used timestamps, and a revoked flag.

## Rotation and revocation

Refresh tokens rotate on every use: each refresh issues a new refresh token
and invalidates the one just used. If an already-rotated-out refresh token is
ever presented again, that's a signal of token theft, and the entire session
is revoked.

The sessions table is only touched at login, refresh, logout, and when
listing active devices — not on every regular request, which is what keeps
per-request access-token verification cheap regardless of whether that
verification happens locally or via network call (see uauth-service § Key
Design Decisions).

A burst of failed refresh attempts against one session is a theft signal
worth monitoring, since valid refresh tokens are high-entropy and not
realistically guessable (see uauth-service § Abuse protection).

## Rotation concurrency and recovery

Two near-simultaneous refresh calls presenting the same token (e.g. a client
retry racing the original request) are resolved with a conditional write:
rotation updates the session's stored refresh-token hash only if it still
matches the presented token, so only one of two concurrent calls can win.
The loser doesn't get treated as theft outright — the *immediately-previous*
refresh token remains acceptable for a short grace window after rotation,
absorbing this exact race (and the equivalent multi-tab race in web) without
false positives. A rotated-out token presented *outside* that grace window is
treated as theft, and the session is revoked. The grace window's duration is
a tuning parameter, not yet chosen (see Open Questions).

If the client never receives the response carrying a freshly-rotated refresh
token (a crash or dropped connection after the server-side write), no
special recovery path exists: the client is left holding a dead refresh
token, and the next refresh attempt fails. This is an acceptable outcome —
the client falls back to a full login, the same behavior as any other
refresh failure.

## Retention

Revoked and expired session records are removed via a DynamoDB TTL
attribute, not an active cleanup job — the record is written with an
expiration timestamp (session max lifetime, or revocation time plus a short
retention window for audit purposes) and DynamoDB garbage-collects it
natively.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Refresh token format | Opaque random string, looked up against the sessions table | Self-contained JWT refresh token | A JWT refresh token can't be revoked before it naturally expires without an explicit revocation check on every use, which erases the "no DB lookup" benefit that motivates using a JWT in the first place. Refresh tokens live far longer than access tokens, so an unrevocable window is a much larger risk here than for access tokens. An opaque token forces a server-side lookup on every refresh, which is acceptable because refresh happens rarely (roughly once per access-token TTL) rather than on every request — and that lookup is also the natural place to add rotation and reuse detection, giving instant, reliable revocation exactly where it matters most. |
| Concurrent-refresh handling | Conditional write + short grace window accepting the immediately-previous token | Treat any reuse of a rotated-out token as theft, no grace window | A hard "any reuse is theft" rule produces false positives from ordinary retries and multi-tab clients, which are expected, benign traffic, not attacks — a short grace window absorbs those while still catching reuse far outside any plausible retry timing. |
| Session record cleanup | DynamoDB TTL attribute (native expiry) | An active/scheduled cleanup job | Native TTL requires no additional infrastructure or scheduled compute, consistent with the project's cheap-ops goal. |

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
