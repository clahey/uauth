---
parent: uauth-service
prefix: SVC-ACCT
---

# accounts

## Context and Design Philosophy

accounts owns the account/identity data model: mapping a login method's
proof of identity to a stable, canonical uauth account, creating a new
account on first login, and looking up an existing one on repeat logins.
This is also where multi-method account linking/unlinking will live once a
second login method exists.

uauth's identity output (a stable, canonical user ID plus standard claims) is
the entire contract downstream consumers depend on (see root HLD § Tenets —
*Minimal, stable identity surface*) — accounts is what makes that ID stable
across repeat logins and, eventually, across linked methods.

## Account creation and lookup

On a successful Google login, login-methods passes the verified Google `sub`
claim to accounts. accounts looks up an existing account by that identifier;
if none exists, it creates one. This is the only identity-provider mapping
in place today, since Google is the only implemented login method.

## Profile data

For now, profile display data (`name`, `picture`, `email`) is a pass-through
of whatever the login provider supplies at login time — accounts does not
yet store or serve custom-settable profile fields (see root HLD § System
Design — Client-facing API).

Account creation uses a conditional write keyed on the identity-provider
mapping (e.g. "create only if no account exists for this Google `sub`
already"), so two near-simultaneous first-time logins with the same new
identity can't both succeed in creating separate accounts — one write wins,
the other fails and re-reads the lookup to find the account the winner just
created.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Concurrent first-time-login account creation | Conditional write on the identity-provider mapping; the losing caller re-reads and uses the winner's account | Unconditional create (accept the race) | An unconditional create would let two near-simultaneous logins for the same new identity produce two separate accounts for one person, silently fragmenting their data — a conditional write makes creation idempotent under concurrency at negligible cost. |

## Open Questions & Future Decisions

### Deferred

1. Account/identity schema (what fields an account record actually holds
   beyond the identity-provider mapping) is not yet designed.
2. Multi-method account linking/unlinking (the Account + LinkedIdentity
   shape referenced in the root HLD's rejected-Cognito-alternative
   discussion) is not yet designed — not needed until a second login method
   exists.
3. Whether/how custom-settable profile fields (e.g. a user-uploaded picture)
   would be added later, and what storage that implies (e.g. S3 + presigned
   upload), is unresolved and out of scope until requested.

## References

- Parent: `docs/intent/uauth-service/uauth-service-design.md`
- Root HLD: `docs/high-level-design.md`
