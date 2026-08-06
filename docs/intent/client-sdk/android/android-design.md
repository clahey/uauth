---
parent: client-sdk
prefix: CLIENT-ANDROID
---

# client-sdk / android

## Context and Design Philosophy

The Android client SDK implements the client-sdk contract in Kotlin for the
Android app (Jetpack Compose UI). It is the first client SDK to reach MVP,
alongside web and server-sdk.

## Google login ceremony

`loginWithGoogle()` uses Google's native SDK (Credential Manager / Google
Identity Services) to present a native account picker and obtain a
Google-signed ID token directly — no browser redirect. This is what makes
Google login "as low-friction as possible" on Android (root HLD § Goals).
The resulting ID token is sent to uauth-service/login-methods for
verification.

## Token storage

The refresh token (the session — see uauth-service/sessions) must be stored
securely between app runs. Not yet decided: EncryptedSharedPreferences vs.
Android Keystore-backed storage directly — see Open Questions.

## Login failure and recovery

Credential Manager returning no credential (no Google account on the device,
or the user cancels the picker) is a normal, expected outcome, not an
exception — `loginWithGoogle()` returns a "not logged in" result the calling
app can present its own retry/empty-state UI for.

If a stored refresh token survives an Android backup/restore or reinstall
onto a new device, but the corresponding session was revoked server-side in
the meantime (see uauth-service/sessions), the resulting "session not found"
response (see login-methods § Verification flow) is treated the same way: a
normal "not logged in" outcome that routes back to the login ceremony, not a
crash or fatal error.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Handling a missing/cancelled credential and a revoked/restored session | Both treated as an ordinary "not logged in" result, not an exception | Throwing/crashing on these paths | Neither case is a bug or an unexpected state — first run without a Google account, a cancelled picker, and a restored install with a since-revoked session are all normal parts of the login lifecycle a calling app needs to handle the same way it handles "never logged in." |

## Open Questions & Future Decisions

### Deferred

1. Secure local token storage mechanism (EncryptedSharedPreferences vs.
   Keystore-backed storage) is not yet chosen.
2. `getToken()`'s transparent-refresh implementation (when to proactively
   refresh vs. refresh on 401, concurrency handling if multiple requests
   need a token at once) is not yet designed.

## References

- Parent: `docs/intent/client-sdk/client-sdk-design.md`
- Root HLD: `docs/high-level-design.md`
