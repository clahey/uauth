# High-Level Design: uauth

## Problem

There is an existing GraphQL-based project whose authentication is tied directly to
its server's own infrastructure — data from that project has never been reachable
from an Android app because of this coupling. More broadly, every authentication
system evaluated so far has been either an assembly of disconnected pieces requiring
custom integration, tied to one particular server structure, or closed source.
uauth exists to find or build a reusable authentication system, decoupled from any
single server, that can serve multiple independent projects across multiple client
surfaces without re-solving login from scratch each time. Authorization — what an
authenticated user is allowed to do — is a related but deliberately separate
concern; see Non-Goals.

## Approach

uauth's scope is authentication only: verifying identity across multiple login
methods and issuing sessions/tokens, exposed through a minimal, stable identity API
(e.g. `getCurrentUser`, login, link/unlink). Authorization is out of scope (see
Non-Goals). Authentication itself is custom and self-orchestrated rather than built
on a managed identity service or an open-core identity server — see *Authentication
mechanism* under Key Design Decisions.

## Target Users

- **Android app users** — access their data via a native Android app (Kotlin,
  Jetpack Compose UI) talking to the business logic server.
- **Website users** — access the same data via a JS/TS web frontend talking to the
  same business logic server.
- **Future client surfaces (anticipated, not committed)** — a desktop app (planned,
  Kotlin via Compose Multiplatform, for low-widget-count UIs) and an iOS app
  (anticipated; UI approach undecided — Compose Multiplatform may not suit
  widget-heavy iOS screens, so a different iOS UI framework is possible). The
  authn design should not preclude adding these later, and client-side
  integration should be workable from Kotlin (Android/desktop) and JS/TS (web) at
  minimum, without assuming a single client language.
- **Consuming projects** — multiple independent projects/servers, built and
  maintained by a solo developer (working with heavy use of AI coding agents),
  each integrating uauth as a shared authentication layer instead of
  reimplementing its own.

## Goals

- Users can log in with an existing Google account ("Sign in with Google"), rather
  than only a system-local username/password.
- The Google login flow on the Android app is as low-friction as possible for the
  user (minimal taps/screens, native-feeling — not a bolted-on browser redirect if
  it can be avoided).
- uauth's own authentication interface — the API it exposes for login, session, and
  identity operations — may use any API paradigm (REST, GraphQL, or other); this is
  not yet decided, and open to whichever fits best. JSON is the likely default
  payload format but is not a hard requirement; a better-fitting format (e.g.
  SOAP/XML) is acceptable if it's genuinely the better choice. This is independent
  of whatever paradigm a consuming project's own business logic server uses toward
  its own clients (see Non-Goals).
- uauth-service (uauth's own backend) runs serverless (e.g. AWS Lambda), not as
  a long-lived server process. AWS is the default target given existing team
  familiarity; reasonable portability to another provider is a soft
  preference, not a hard requirement — don't sacrifice AWS-native fit to chase
  portability. This constrains uauth's own architecture only — see Non-Goals
  for why it doesn't extend to how a consuming project runs its business logic
  server.
- OIDC providers beyond Google, OTP, traditional username/password login, and
  passkeys are future scope — Google login is the only login method being
  built now. This isn't optional polish: the architecture must be able to
  support adding these methods later without a redesign, which constrains
  current design work even though none of them are implemented yet (see
  uauth-service/login-methods for how each is currently scoped).
- uauth itself is intended to be open source (not just built on open-source
  dependencies) — worth designing with a general-purpose, publishable API rather
  than only what's minimally needed for one project.
- Users can link and unlink multiple authentication methods (e.g. Google, another
  OIDC provider, password) to a single account.
- Clients authenticate to the (stateless, serverless) backend using
  short-lived JWT access tokens as bearer tokens. This is distinct from the
  session itself: refresh tokens (what a login actually produces and what
  persists between app runs) are opaque, not JWTs — see
  uauth-service/sessions for the full access/refresh split and why they're
  shaped differently.
- uauth's identity output (a stable, canonical user ID plus standard claims) is
  sufficient on its own for a consuming project to build any authorization model
  it needs, including sharing/collaboration between users (not strict
  single-owner-only access) — different projects are expected to make different
  authorization choices.
- Infrastructure is manageable as code (e.g. AWS CDK, Amplify) so ops stays cheap
  and deployment can run entirely through CI/CD with minimal manual operation.
- When choosing a hosted service, prefer one that hosts genuinely open-source
  server software over a fully closed-source hosted service — this preserves the
  option to self-host later if needed (see Non-Goals, Tenets).

*(requirements gathering in progress — more to come)*

## Non-Goals

- Running uauth-service itself as a traditional always-on process (e.g. a
  long-lived container or VM-hosted server) is out of scope (see Goals). This
  doesn't constrain how a consuming project runs its own business logic
  server — uauth's verification API is called via IAM/SigV4, which any
  AWS-credentialed caller can do whether it's a Lambda or a traditional
  long-running server; a consuming project's architecture is its own choice
  (see *Building or maintaining any individual consuming project's business
  logic server* below).
- Guaranteed portability across cloud providers is not a requirement. Code that is
  reasonably reusable elsewhere is a bonus, not a design driver.
- Role-based / org / team / admin-hierarchy authorization is not needed by any
  currently-imagined consuming project — don't build for it speculatively.
- Migrating or transferring existing users from the current GraphQL project onto
  uauth is not required; a clean cutover (new accounts) is acceptable.
- Building or maintaining any individual consuming project's business logic server
  (e.g. the existing GraphQL project) is out of scope of uauth itself — uauth is
  the shared authentication layer those servers integrate with. uauth is agnostic
  to that server's own client-facing API paradigm (REST, GraphQL, or otherwise);
  like authorization, that choice belongs entirely to each consuming project.
- Implementing, bundling, or providing an integration API for any authorization
  mechanism (a relationship/policy engine, or hand-rolled checks) is out of scope.
  Authorization is each consuming project's own decision, built directly on
  uauth's standard identity output — no uauth-side authorization API, adapter, or
  default implementation is provided. (See *Authn/authz interaction* under
  Options Under Consideration for the reasoning.)
- Building or maintaining a DB-structure-enforcing data-access layer (e.g. a
  mandated schema plus automatic sharing-constraint checks baked into read/write
  primitives like a `readDBRow` helper) is out of scope of uauth. This is a
  genuinely interesting idea for a *separate*, potential future project that
  would consume uauth's identity output — not something uauth builds toward or
  exposes an integration API for.
- MFA is out of scope for OIDC logins — the OIDC provider (e.g. Google) is trusted
  to own that. Revisit MFA once password/OTP login is real, since those methods
  don't get MFA for free from a third party.
- Account recovery flows (e.g. forgot-password) are out of scope until
  password/OTP login is real; not needed while OIDC is the only login method.
- Depending on closed-source software anywhere in the client (Android app or
  website) is out of scope, without exception.
- Depending on a fully closed-source vendor/server (for authentication or
  otherwise) is out of scope, with the AWS-native-primitive exception defined
  in Tenets.
- Paying a vendor add-on with a high fixed monthly minimum (e.g. a $100/month
  floor per feature) is unacceptable, regardless of how good the feature
  otherwise is — this rules out per-feature add-ons priced that way (e.g. paid
  account-linking or MFA tiers), not just the vendor overall.

## Tenets

- **Security first, ease of use second.** When a choice trades off user friction
  against security, security wins — but ease of use is still a strong, explicit
  second priority, not an afterthought.
- **Open source over closed, narrowly excepted.** Prefer open-source software. The
  exception is scoped to AWS-native infrastructure *primitives* (Lambda, DynamoDB,
  S3, IAM, etc.) used underneath something we build ourselves — those are accepted
  as the infrastructure baseline regardless of license. A full off-the-shelf
  product — AWS-provided or third-party (e.g. Cognito) — does not get that
  exception automatically; it's weighed as a vendor dependency like any other.
  Closed source in the client is never acceptable, no exceptions.
- **Minimal, stable identity surface — no bespoke integration APIs.** uauth's
  contract with anything built on top of it — an authorization check, a future
  companion library, anything else — is its standard identity API (e.g.
  `getCurrentUser`, login, link/unlink) and nothing more. If a downstream use
  case seems to need *extra* uauth-side API surface beyond that standard
  identity surface, treat that as a signal the boundary is drawn wrong, not a
  feature request to fulfill.

## Options Under Consideration

Supporting comparison behind the authentication mechanism decision recorded under
Key Design Decisions, kept here as the detailed backing analysis and as reference
material for authorization engines, which remain each consuming project's own
choice.

### Authentication comparison table

| Dimension | Option A — Cognito | Option B — SuperTokens | Option C — Custom authn |
|---|---|---|---|
| Authn architecture | AWS-native managed identity service (User Pools). Native low-friction Android Google login requires bypassing the Hosted UI via a custom auth flow (`DefineAuthChallenge`/`CreateAuthChallenge`/`VerifyAuthChallengeResponse` Lambda triggers) rather than the console-configured default. | Open-core: frontend SDK + backend SDK (Lambda-friendly, embeds in app server) + a separate **Core** service that does the real auth logic/DB work. | Self-orchestrated: call Google's native Android SDK directly for the ID token (no Hosted-UI-style default to bypass), verify it with an off-the-shelf JWT/JWKS library, issue our own JWTs (access + refresh) via a library + a KMS-managed signing key, own the user/account data model in DynamoDB from day one. |
| License | Fully closed source (AWS proprietary product). Not covered by the AWS-native-primitive tenet exception (see Tenets) — evaluated as a vendor dependency. | Apache 2.0 for the base; a proprietary license applies to specific add-on features (`ee/` dir). Open-core, not fully open. | Our own code plus narrowly-scoped open-source libraries (JWT verification, WebAuthn, password hashing). No vendor core, no open-core split, no proprietary tier ever in play. |
| Serverless/Lambda fit | Fully Lambda-native; no extra infrastructure to run. | Backend SDK is Lambda-friendly, but the Core is a persistent service — not Lambda-shaped. Self-hosting it means an always-on container (Fargate/ECS/VM); avoiding that means using SuperTokens' *hosted* Core (a vendor dependency again). | Cleanest fit of the three — pure Lambda functions + DynamoDB + KMS, all accepted AWS-native primitives under the Tenets. No Hosted UI, no persistent Core service. |
| Account linking | Not a console default; requires custom `PreSignUp`/`PreAuthentication` Lambda triggers — but Cognito provides a real merge primitive (`AdminLinkProviderForUser`) to build on. | Free tier has **no merge primitive at all** — the packaged Account Linking feature is a paid add-on ($0.01/MAU, $100/month minimum — ruled out per Non-Goals). DIY means building the entire identity-resolution/session-remapping layer from scratch on top of independent per-method user records — more work and more account-takeover risk surface than Option A's DIY path, since there's no supported linking primitive to lean on. | No retrofit problem — the Account + LinkedIdentity schema (one Account row, many LinkedIdentity rows keyed by provider) is designed correctly from day one. Roughly the same amount of code as Option B's DIY layer would need anyway, minus the fight against a platform that assumes one-user-per-method by default. |
| MFA | Native TOTP/SMS MFA, console-configurable. | Paid add-on ($0.01/MAU, $100/month minimum — ruled out). Cheap and low-risk to DIY instead: TOTP (RFC 6238) is a small, standardized algorithm with mature open-source libraries in every language, no identity-merging ambiguity, and the free `usermetadata` recipe is sufficient for secret storage. DIY recommended here regardless of which option we pick. | Same DIY TOTP approach as Option B — a wash across all three options; no vendor tier to avoid here since there's no vendor. |
| Passkeys | Native WebAuthn support exists. Bulk/admin public-key export API not found — likely means re-registration is required on any future migration, similar to the password-hash lock-in pattern. Unconfirmed, not blocking. | Docs exist for a passkeys recipe; which pricing tier it falls under is unconfirmed. | `@simplewebauthn/server` (TypeScript, actively maintained, v13.x as of 2026) or an equivalent library in another language handles the WebAuthn ceremony. We own storage of the public key and relying-party config — no vendor pricing/export uncertainty, but no vendor-provided flow orchestration either. |
| Credential portability | Password hashes: not exportable (proprietary SRP scheme) — irrelevant today since Google OIDC is the only login method. OIDC/federated users are **not** locked in: profile data is exportable via admin APIs, users just re-auth with Google elsewhere. | Not yet researched — would need the same check if the EmailPassword recipe is ever used. | Not applicable in the lock-in sense — all credential material (refresh tokens, TOTP secrets, WebAuthn public keys, password hashes if added) lives in our own DynamoDB tables, so there's nothing vendor-held to export. |
| Vendor-lock shape | One vendor (AWS) for the whole authn piece; evaluated as a full product, not exempted by AWS-native-primitive status. | Genuinely open source at the core (self-hostable, real escape hatch) — but the two features this project most wants (linking, MFA) sit behind a paid tier with an unacceptable price floor, narrowing the practical benefit of "open source" for this project's actual needs. | No authn vendor at all. Trades lock-in risk for full ownership of security-critical orchestration (token issuance, refresh rotation, revocation) — libraries handle the cryptography, we own correctness of how they're wired together, with no vendor/security-team backstop on that wiring. |

### Option C — candidate component libraries ("glue" inventory)

- **Google ID token verification** — any JWKS/JWT library (e.g. `jose` for Node/TS) validating signature + `iss`/`aud`/`exp` against Google's public keys. Well-trodden, low risk.
- **Own JWT issuance** (access + refresh tokens) — same class of library, signing with an asymmetric key held in AWS KMS.
- **Passkeys/WebAuthn** — `@simplewebauthn/server` (TypeScript; actively maintained, v13.x as of 2026) or an equivalent WebAuthn library in another language.
- **Password hashing** (if/when added) — Argon2 via a maintained library in whichever language is chosen.
- **OTP delivery** (if/when added) — SES (email) or SNS (SMS) for delivery; a DynamoDB table with a TTL attribute for the one-time code.
- **Session/refresh-token storage and revocation** — a DynamoDB table tracking active sessions per device, checked on each request. Needed regardless of which option is chosen if per-device revocation ("kill my old phone's session from my laptop") is wanted, since it's unconfirmed whether Cognito or SuperTokens expose that granularity natively.

### Comparative effort read

Comparing the three by the specific flows this project needs, rather than by generic defaults, since Options A and B both carry real custom-code burden of their own for those flows:

- **Native Android Google login**: Option A requires bypassing Cognito's Hosted UI via custom-auth-flow Lambda triggers; Option C just calls Google's native Android SDK directly and verifies the token with a library — there's no default flow to bypass. Comparable to or less work than Option A, only marginally more than Option B's built-in recipe.
- **Account linking**: this is where Option C compares best. It was the most expensive/risky item under Option B (retrofitting a merge concept onto a platform with no merge primitive) and required Lambda-trigger glue under Option A. Under Option C it's just the schema designed from day one — no retrofit, no fighting a platform default.
- **MFA**: a wash — cheap DIY TOTP under all three options.
- **Passkeys**: comparable effort to A/B once their vendor-specific open questions (Cognito's unconfirmed export API, SuperTokens' unconfirmed pricing tier) are counted as real integration risk rather than assumed-free.
- **What Option C uniquely costs**: token issuance, refresh rotation, and revocation become our responsibility end-to-end — library help on the cryptography, but no vendor/security-team backstop on the orchestration logic. That is the genuine tradeoff, not "reimplementing cryptography," which the library ecosystem already covers.

Net: Option C's *incremental* custom-code burden over what A or B already force onto us (Lambda-trigger-shaped glue either way, for the Android and linking flows specifically) is modest — see *Authentication mechanism* under Key Design Decisions for the full decision and rationale.

### Authorization engine reference notes

Reference material only, not a uauth decision (see Non-Goals) — useful to a
consuming project picking its own authorization approach on top of uauth's
identity output.

| Dimension | OpenFGA | Casbin | Cedar (embedded) | Hand-rolled |
|---|---|---|---|---|
| Architecture | Separate persistent server + relational datastore (Postgres/MySQL); `Check` API called over the network. | In-process library, no separate server; policy store can be anything we provide (e.g. our own DynamoDB). | In-process library (`cedar-policy` Rust crate); no separate server if self-embedded rather than using AWS Verified Permissions (the hosted, non-open path). | In-process application code; no library at all. |
| License | Apache 2.0, CNCF-hosted, fully-featured open source — no paywalled tier (unlike SuperTokens). | Apache 2.0. | Apache 2.0 (AWS-originated). | N/A — our own code. |
| Model expressiveness | Purpose-built for relationship-based access (ReBAC) — hierarchies like "viewer of folder implies viewer of documents inside" out of the box. Most naturally matches the sharing/collaboration Goal. | Supports ACL/RBAC/ABAC and, per current docs, ReBAC-style patterns too, via a config-driven model + matcher expressions. Less naturally hierarchical than OpenFGA by default. | Own policy language, designed for readability and formal verification of policy properties; supports RBAC/ABAC and hierarchy via its schema. | Full expressiveness — but we own discovery of every edge case (revocation, negative rules, hierarchy) ourselves. |
| Serverless fit | Poor — needs an always-on container (Fargate/ECS) or a hosted offering (Okta FGA), same shape as Option B's Core-service concern. | Excellent — runs inside the same Lambda function, no extra infrastructure. | Excellent — same as Casbin, runs in-process. | Excellent — it's just our code. |
| Multi-project reuse | Native `Store` concept — one deployment serves many isolated per-project authorization models. | Reuse is the shared library + a per-project policy/config file; no shared running deployment, but the engine itself is reusable. | Same shape as Casbin — reusable engine, per-project policy files. | Weakest reuse story — logic tends to accumulate project-specific special cases unless deliberately factored into a shared internal library. |
| Language fit for this stack | N/A (network API, language-agnostic). | Broadest language coverage of the embedded options (Go, Java, Node, Python, more) — fits regardless of what language the Lambda backend ends up in. | Native Rust; maturity of bindings for other languages (JVM/Kotlin, Node) not yet confirmed. | N/A — whatever language the backend is already in. |

For a project favoring no persistent service, the embedded options (Casbin,
Cedar, hand-rolled) are the structurally better fit; OpenFGA needs to justify
an exception (a self-hosted Fargate service, or a hosted vendor dependency
like Okta FGA).

### Authn/authz interaction

uauth is authentication-only. Authentication and authorization are nearly
fully separable: authn's job ends at producing a verified, stable identity;
authz decides what that identity can do, per request, against live
relationship/policy data that can't be precomputed at login time anyway,
since sharing relationships change after a session is already issued. The one
real coupling point is that authn must produce a single, stable, canonical
user identifier per person, consistent across every linked login method — any
authorization approach a consuming project picks keys its rules or tuples off
that identifier, so inconsistent linking would fragment a project's
authorization data (permissions granted under one linked identity invisible
under another). That contract is owned entirely by uauth's account-linking
design. It doesn't benefit from uauth also implementing authorization: the
actual integration is a plain identifier comparison (e.g.
`enforce(user.id, resource, action)`), not a translation step where bundling
could reduce risk.

An adapter layer per authorization engine wouldn't add anything beyond what a
project already gets by passing `user.id` directly into a check — there's no
real translation surface for an adapter to own. A bundled default
implementation for the common ownership/sharing case would need to know a
consuming project's resource schema and field names to be genuinely
zero-setup, which reaches into that project's business logic (out of scope),
and would work against different projects needing different authorization
approaches (see Goals).

A related idea — a data-access layer with a mandated schema and automatic
sharing-constraint checks (see Non-Goals) — would, as its own project, need
to own DB schema and query conventions with its own release cadence and
opt-in audience, distinct from uauth's.

### Open questions on these options

- Cognito: does an admin/bulk API exist to export registered passkey public keys? (Not found so far — treat as probably-no until confirmed.)
- SuperTokens: which pricing tier covers passkeys?
- uauth's own authentication-interface API paradigm (REST, GraphQL, or other) has not been explored yet.
- Session/device-revocation granularity ("kill my old phone's session from my laptop") not yet confirmed as native to Cognito or SuperTokens — may be a wash across all authn options rather than a differentiator.
- Cedar's language-binding maturity outside Rust (JVM/Kotlin, Node) is unconfirmed.

Sources referenced while researching these options:
- [list-web-authn-credentials](https://docs.aws.amazon.com/en_us/cli/latest/reference/cognito-idp/list-web-authn-credentials.html)
- [supertokens/supertokens-core](https://github.com/supertokens/supertokens-core)
- [SuperTokens pricing](https://supertokens.com/pricing)
- [@simplewebauthn/server](https://www.npmjs.com/package/@simplewebauthn/server)
- [OpenFGA (GitHub)](https://github.com/openfga)
- [Casbin vs. Cedar comparison](https://slashdot.org/software/comparison/Casbin-vs-Cedar/)
- [cedar-policy (crates.io)](https://crates.io/crates/cedar-policy)

## System Design

uauth is three top-level components, each detailed in its own design subtree
under `docs/intent/`:

- **uauth-service** — the backend: verifies proof of identity per login
  method, issues and manages sessions, owns the account/identity data model,
  and exposes it through two API surfaces (a public client-facing surface,
  and an IAM-authenticated server-to-server verification surface). See
  `docs/intent/uauth-service/uauth-service-design.md` and its children
  (login-methods, sessions, accounts).
- **client-sdk** — the untrusted client libraries (Android/Kotlin, web/TS)
  implementing the shared login/logout/getCurrentUser/getToken contract. See
  `docs/intent/client-sdk/client-sdk-design.md` and its children (android,
  web).
- **server-sdk** — the per-language library (Python first) a consuming
  project's business logic server uses to verify identity on incoming
  requests. See `docs/intent/server-sdk/server-sdk-design.md`.

## Key Design Decisions

### Authentication mechanism: custom, self-orchestrated

uauth authenticates users via directly-orchestrated flows built from narrowly-scoped
open-source libraries — Google ID token verification via a JWKS/JWT library,
self-issued access-token JWTs signed with a KMS-managed key (refresh tokens are
opaque, not JWTs — see *Refresh tokens are opaque, not JWTs* below), WebAuthn via
`@simplewebauthn/server` or an equivalent library, TOTP via a standard library —
rather than adopting a managed identity service or an open-core identity server.
uauth owns the account/session data model in DynamoDB from day one, including the
Account + LinkedIdentity schema that multi-method account linking needs.

**Alternatives considered:**

- **AWS Cognito** — rejected. Fully closed source and not covered by the
  AWS-native-primitive exception in Tenets, since it's a full off-the-shelf product
  rather than infrastructure underneath something we build. Low-friction native
  Android Google login also requires bypassing Cognito's Hosted UI default via
  custom auth-flow Lambda triggers — comparable custom-code burden to the
  self-orchestrated approach, without its license or serverless-purity benefit.
- **SuperTokens** — rejected. Its Core service is a persistent process, not
  Lambda-native, so self-hosting it means running an always-on container; using
  SuperTokens' hosted Core reintroduces a vendor dependency either way. More
  decisively, the two features this project most needs — account linking and MFA —
  are paid add-ons with a $100/month price floor, ruled out by Non-Goals. Working
  around that limitation means building the entire identity-resolution/
  session-remapping layer on top of a platform that assumes one-user-per-method by
  default — more work and more account-takeover risk surface than building the same
  layer against a clean schema.

**Rationale:** this is the only option that satisfies both the serverless/
Lambda-native goal and the "uauth itself is open source" goal without caveats, and
the only one where account linking is designed correctly from day one rather than
retrofitted onto a platform or vendor add-on. The genuine cost is that uauth owns
token issuance, rotation, and revocation correctness end-to-end, with no vendor
security team backstopping that orchestration — every other flow (Google login,
MFA, passkeys) draws on mature, well-trodden libraries rather than novel
cryptography.

This decision is reversible if custom orchestration proves too costly to maintain:
consuming projects depend only on uauth's standard identity API (see *Minimal,
stable identity surface* Tenet), so the backing mechanism could migrate to Cognito
or SuperTokens later without changing what downstream projects integrate against.

Two decisions that follow directly from this one — how business servers verify
identity (network call vs. local verification), and why refresh tokens are
opaque rather than JWTs — are recorded where their substance lives: see
`docs/intent/uauth-service/uauth-service-design.md` § Key Design Decisions and
`docs/intent/uauth-service/sessions/sessions-design.md` § Decisions &
Alternatives.

## Success Metrics

*(not yet specified)*

## References

*(not yet specified)*
