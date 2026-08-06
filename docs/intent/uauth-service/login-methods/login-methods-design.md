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

Login is exposed as one concrete client function per supported method (e.g.
`loginWithGoogle()`) rather than a single generic, pluggable `login(method,
...)` entry point, since the set of supported methods is closed and owned by
uauth itself — not something a consuming project supplies its own
implementation of. Verification of each method's proof is uauth's own
internal strategy per method (JWKS check, password hash check, OTP check,
WebAuthn verification); this is an implementation detail, not exposed
integration surface.

## Supported methods

### Google (OIDC) — implemented for MVP

The client does not talk to uauth first. On Android, Google's native SDK
(Credential Manager / Google Identity Services) presents a native account
picker and returns a Google-signed ID token directly, no browser involved; on
web, the standard OIDC redirect flow (authorization code exchanged for
tokens) applies instead. Either way, the client sends the resulting Google ID
token to uauth. login-methods verifies the token's signature against
Google's public JWKS and checks `iss`/`aud`/`exp` before requesting a new
session from sessions. On success, it looks up or creates the corresponding
account (see accounts) keyed by Google's `sub` claim, and passes basic
profile claims (`name`, `picture`, `email`) through for display — no
custom-settable profile fields yet (see accounts).

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

## Verification flow

On any method's successful proof-of-identity check, login-methods first calls
accounts to look up or create the corresponding account, then calls sessions
to issue a new session. Account lookup-or-create is idempotent (safe to
repeat with the same identity-provider mapping — see accounts), so if session
issuance fails after account creation succeeded, the client simply retries
the whole login call: it will find the already-created account and proceed
normally. No rollback of account creation is needed.

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

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Client login surface shape | One concrete function per supported method (`loginWithGoogle()`, etc.) | A single generic, pluggable `login(method, ...)` entry point | The set of supported methods is closed and owned by uauth, not extended by consuming projects — a generic pluggable entry point implies extensibility that doesn't exist. |

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

## References

- Parent: `docs/intent/uauth-service/uauth-service-design.md`
- Root HLD: `docs/high-level-design.md`
