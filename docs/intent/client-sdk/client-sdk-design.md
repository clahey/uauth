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

- One concrete function per supported login method (e.g. `loginWithGoogle()`;
  see uauth-service/login-methods for why this is per-method functions
  rather than a generic, pluggable `login(method, ...)` entry point), plus
  `logout()`.
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

## Children

- **android** — Kotlin, for the Android app.
- **web** — TypeScript, for the JS/TS web frontend.

Future client surfaces (desktop via Compose Multiplatform, iOS — UI approach
undecided) are anticipated but not yet in scope (see root HLD § Target
Users).

## Open Questions & Future Decisions

### Deferred

1. Whether the shared contract above should be formalized as a
   platform-agnostic interface definition (e.g. for code-generation) or
   stays as independently-implemented parallel libraries per platform.

## References

- Root HLD: `docs/high-level-design.md`
