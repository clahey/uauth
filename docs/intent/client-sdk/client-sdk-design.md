---
parent: high-level-design
prefix: CLIENT
---

# client-sdk

## Context and Design Philosophy

client-sdk is the shared contract every uauth client library implements,
regardless of platform. Client SDKs run as untrusted code from the server's
perspective — nothing they assert about identity is trusted directly; the
business logic server always verifies independently through server-sdk. Each
platform (android, web) owns how it satisfies this contract; this document
covers the contract itself.

## Shared contract

- One concrete function per supported login method (e.g. `loginWithGoogle()`),
  plus `logout()`. See Key Design Decisions below for why this is per-method
  functions rather than a generic, pluggable `login(method, ...)` entry
  point.
- `getCurrentUser()` — returns display information for the current user. For
  now this is a pass-through of whatever the login provider supplies (e.g.
  Google's `name`, `picture`, `email` claims) — no custom-settable profile
  fields yet. Token access is a separate accessor (e.g. `getToken()`), not a
  static field on the returned user object, since access tokens are
  short-lived and must be fetched fresh at the point of use, refreshing
  transparently under the hood as needed — this means each platform's SDK
  needs real token-refresh logic and secure local storage for the refresh
  token between app runs, not just the login ceremony itself.
- Link/unlink authentication methods on the current account (per root HLD §
  Goals; shape not yet elaborated — depends on uauth-service/accounts design
  work not yet done).
- A convenience authenticated-HTTP-request helper that attaches the current
  token to an outgoing request automatically is a secondary, nice-to-have
  addition — a thin wrapper over the token accessor plus the platform's
  native HTTP client, not a uauth-owned primitive or a single cross-platform
  implementation.

## Interface

Language-agnostic contract; see android and web for the idiomatic signature
in each platform's language.

| Function | Params | Returns | Notes |
|---|---|---|---|
| `loginWithGoogle` | none | Success: a `CurrentUser`. Failure: nothing — a "not logged in" outcome, not an exception (see android § Login failure and recovery for the reasoning, which applies to web too). | See Key Design Decisions above for why this is a dedicated function per method. |
| `logout` | none | nothing | Clears local state regardless of whether the server call succeeds. |
| `getCurrentUser` | none | a `CurrentUser` if logged in, otherwise nothing | Synchronous — reads profile data cached at login, no network call. |
| `getToken` | none | a valid access token string if logged in, otherwise nothing | Asynchronous — refreshes transparently if the cached access token is stale. Returns nothing if the session is dead (refresh failed), signaling the caller to treat the user as logged out. |

`CurrentUser` shape: `{ userId: string, name?: string, picture?: string, email?: string }` — the pass-through provider profile claims described in Shared contract above.

Link/unlink and the optional HTTP-request-wrapper convenience aren't
specified yet — see Shared contract and Open Questions.

## Children

- **android** — Kotlin, for the Android app.
- **web** — TypeScript, for the JS/TS web frontend.

Future client surfaces (desktop via Compose Multiplatform, iOS — UI approach
undecided) are anticipated but not yet in scope (see root HLD § Target
Users).

## Key Design Decisions

### Client login surface shape: one function per method, not a generic entry point

The client SDK exposes one concrete function per supported login method
(`loginWithGoogle()`, etc.) rather than a single generic, pluggable
`login(method, ...)` entry point.

**Alternatives considered:**

- **Generic pluggable `login(method, ...)` entry point** — rejected. This
  shape implies a consuming project could supply its own login method
  implementation, but the set of supported methods is closed and owned by
  uauth itself (see uauth-service/login-methods) — there's nothing to plug
  in.

**Rationale:** matching the actual extensibility model (none) keeps the
client API honest about what it does — a generic entry point would invite an
integration pattern that doesn't exist.

## Open Questions & Future Decisions

### Deferred

1. Whether the shared contract above should be formalized as a
   platform-agnostic interface definition (e.g. for code-generation) or
   stays as independently-implemented parallel libraries per platform.

## References

- Root HLD: `docs/high-level-design.md`
