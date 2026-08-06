---
parent: high-level-design
prefix: SRVSDK
---

# server-sdk

## Context and Design Philosophy

server-sdk is the small per-language library a consuming project's business
logic server uses to verify identity on incoming requests. Python is the
first (and, for MVP, only) supported language. It exposes a stable interface
— `getCurrentUser(headers)` or equivalent — that calls uauth-service's
server-to-server verification surface (see uauth-service § API surfaces) and
returns verified identity for the request. The network-vs-local
implementation choice behind this interface, and why IAM/SigV4 is a
low-cost choice for Lambda-native callers, are recorded in uauth-service §
Key Design Decisions, not repeated here — this leaf owns the library's
surface and behavior, not the backend architecture it calls into.

## Interface

`getCurrentUser(headers)` takes the incoming request's headers (containing
the bearer access token), signs and sends a verification request to
uauth-service via the AWS SDK's SigV4 signing (available for free since the
business server itself runs as a Lambda with its own execution-role
credentials — see uauth-service § Key Design Decisions), and returns verified
identity for the request.

A missing `Authorization` header and an invalid one (malformed, expired, or
failing signature/claims checks) both return the same "no identity" result
(e.g. `None`/`null`) rather than being distinguished at this library's
surface — callers generally only need to know "is there a verified user or
not," and collapsing the two cases keeps the interface simple, consistent
with the project's minimal-surface tenet. uauth-service's own logs still
distinguish them for monitoring purposes; this library just doesn't surface
that distinction to callers.

The verification call uses a short timeout and no automatic retry. If the
call times out or otherwise fails (uauth-service unreachable, throttled,
erroring), `getCurrentUser` fails closed — it raises rather than returning
"no identity" — so a caller can't mistake "verification was unavailable" for
"this request is legitimately unauthenticated." The exact timeout value is
not yet chosen (see Open Questions).

Caching a verification result is not implemented for now. It would be safe
to do — a cache entry keyed by the token, expiring no later than the token's
own `exp` claim, cannot return a stale answer, since an access token's
validity doesn't change before it expires (there's no per-access-token
revocation check in this design; see uauth-service/sessions) — but it's not
worth designing until there's an actual latency or cost problem to solve.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Missing vs. invalid token | Both return the same "no identity" result | Distinguish with different return values/exceptions | Callers need "is this request authenticated," not the specific reason it isn't — a finer-grained surface would be unused complexity, contrary to the minimal-surface tenet. |
| Verification-call failure (timeout, uauth-service unreachable) | Fail closed (raise) | Fail open (treat as "no identity") | Silently treating an unreachable verification service as "unauthenticated" would let real requests get rejected as if the user were logged out, and — worse — could mask a broader outage as a wave of individual auth failures instead of a surfaced dependency problem. |
| Verification-result caching | Not implemented | Cache with TTL bounded by the token's `exp` | Would be safe to add later (see body text above), but there's no known latency/cost problem to justify the added complexity yet — premature optimization. |

## Open Questions & Future Decisions

### Deferred

1. Whether `getCurrentUser` returns full profile data or only a verified
   identity (user ID + claims) is not yet resolved (tracked in
   uauth-service/sessions § Open Questions, since it affects whether
   verification needs a DB read).
2. The exact timeout value (and whether any retry, e.g. a single retry on
   transient network failure, is worth adding) for the verification call is
   not yet chosen.
3. Which languages beyond Python get their own server-sdk library, and when,
   is not yet decided — this leaf may later promote to a sub-HLD with one
   child per language if that accumulates real per-language complexity.

## References

- Root HLD: `docs/high-level-design.md`
