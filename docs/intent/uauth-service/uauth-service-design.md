---
parent: high-level-design
prefix: SVC
---

# uauth-service

## Context and Design Philosophy

uauth-service is uauth's backend: it verifies proof of identity for each
supported login method, issues and manages sessions, owns the account/
identity data model, and exposes all of this through two distinct API
surfaces. Its three children — login-methods, sessions, and accounts —
divide that work; this document covers how they fit together and how the
service is reached from the outside. It owns no EARS of its own; each child
leaf owns the specs for its own area.

## API surfaces

uauth-service exposes two distinct API surfaces behind the same underlying
implementation. Which surface a given route belongs to is enforced at the API
Gateway route level (authorization type is set per method), not by splitting
into separate Lambda functions — though the implementation may still be
organized as multiple functions sharing common code.

- **Public, client-facing** — login, logout, refresh, profile read. Called
  directly by untrusted clients (mobile apps, browsers), which hold no AWS
  credentials, over plain HTTPS with no IAM authorization. See client-sdk for
  the client side of this surface.
- **Server-to-server verification** — called by a consuming project's
  business logic Lambda to check a request's bearer token. IAM-authenticated;
  see *Server-side identity verification* under Key Design Decisions. See
  server-sdk for the client side of this call.

A consuming project's business logic server has no ambient session — identity
is derived per request from the bearer token in the incoming request's
headers. Each supported server language's library (server-sdk) calls the
verification surface and returns verified identity for the request.

## Abuse protection

The two surfaces need different protection:

- The public surface is uauth-service's own direct front door — no business
  server sits in front of it to absorb traffic — so it needs its own
  throttling (API Gateway usage plans, optionally AWS WAF). For Google-only
  login, the credential material itself (a Google-signed ID token) isn't
  guessable, so the exposure is cost/DoS from request volume rather than
  credential stuffing, and throttling covers it. Password login, once added,
  needs real brute-force defenses on top of that (per-account lockout/
  backoff, not just per-IP rate limiting, since attacks distribute across
  IPs). Refresh tokens are high-entropy opaque strings and aren't
  brute-forceable, but a burst of failed refresh attempts against one
  session is a theft signal worth monitoring (see sessions).
- The verification surface sits behind each consuming project's business
  logic server, which must already protect its own public API from abuse
  regardless of uauth's design — that protection also caps how much traffic
  can ever reach uauth's verification endpoint via a relay, since a
  legitimate, authorized business server will relay whatever it receives,
  including an attacker's garbage tokens. IAM authorization on this route
  doesn't add meaningful abuse protection on top of that; its value is
  identifying the calling project, which lets uauth apply a per-project usage
  plan to contain one project's traffic (malicious or buggy) from degrading
  verification capacity for every other project.

## Children

- **login-methods** — verifies proof of identity for each supported login
  method and hands off to sessions for token issuance.
- **sessions** — access/refresh token issuance, the session record, rotation,
  revocation, and device management.
- **accounts** — the account/identity data model: identity-provider-to-
  account mapping, account creation/lookup, and (future) linking/unlinking
  multiple login methods to one account.

## Key Design Decisions

### Server-side identity verification: network call, not local verification

Business logic servers verify identity via a network call to uauth-service's
verification API, rather than each server verifying JWTs locally/in-process
(e.g. fetching and caching uauth's public JWKS and checking signatures
itself). This sits behind each language's `getCurrentUser(headers)`-shaped
library (see server-sdk), so the network-vs-local choice is an internal
implementation detail, not part of the contract callers depend on.

The verification endpoint runs as a Lambda behind an API Gateway **REST API**
(not HTTP API), authenticated via IAM (SigV4). REST APIs support cross-account
IAM authorization through resource policies that name a calling account or
role directly; HTTP APIs don't support this and would need an extra
`sts:AssumeRole` hop for a consuming project in a different AWS account to
call in. Both uauth-service's verification Lambda and each consuming
project's business logic Lambda stay independently serverless — this
introduces a network hop, not an always-on process.

IAM authorization here identifies the calling project rather than
establishing trust in its claims — verification correctness depends only on
the bearer token presented, not on who's asking. What caller identification
buys is a per-project usage-plan throttle, containing one project's traffic
(malicious or buggy) from degrading verification capacity for every other
project (see Abuse protection above).

**Alternatives considered:**

- **Local/in-process JWKS verification** — not rejected outright, but not the
  default. Its advantages are real: no per-request latency, no live
  availability dependency on uauth, and graceful degradation if uauth has an
  outage (a cached public key keeps working). Its cost is that it requires a
  maintained JWT/JWKS verification implementation in every language a
  consuming project happens to use, which multiplies uauth's own maintenance
  surface across independent projects — the opposite of the "one
  implementation" goal. Because it sits behind the same stable interface, it
  remains available as a per-project escape hatch later if the network-call
  cost proves to be a real problem for a specific project, without requiring
  a contract change for its callers.
- **HTTP API instead of REST API** — rejected for the verification endpoint
  specifically, because HTTP APIs don't support resource-policy-based
  cross-account IAM authorization.

**Rationale:** a single, centrally-maintained verification implementation is
more valuable than per-language reimplementations, especially for a solo-
maintained set of independent projects built with heavy AI-agent use, where
duplicated verification logic per language is duplicated risk with no one
reviewing it closely. This also benefits from concurrency concentration:
because identity checks are fast relative to typical business-logic work, the
verification tier needs far fewer concurrent warm instances than the
business-logic tier calling it, so whatever it caches — signing keys, hot
lookups — stays warmer than if the same cache were fragmented cold across
every business Lambda. The accepted cost is that uauth's own operational
health (a bad deploy, throttling, a DB-side issue, cross-account IAM
misconfiguration) becomes a live, shared dependency for every consuming
project's requests at once — judged an acceptable risk at this project's
solo-maintained, modest scale, especially given the interface leaves room to
swap in local verification per project later if that judgment turns out to
be wrong.

## Open Questions & Future Decisions

### Deferred

1. Whether the public and verification surfaces are implemented as one
   Lambda function or several sharing common code — an implementation/
   deployment choice, not yet made.
2. Infrastructure-as-code structure (CDK stack organization, environment/
   deployment strategy) is not yet designed.

## References

- Root HLD: `docs/high-level-design.md`
