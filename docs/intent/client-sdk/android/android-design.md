---
parent: client-sdk
prefix: CLIENT-ANDROID
---

# client-sdk / android

## Context and Design Philosophy

The Android client SDK implements the client-sdk contract in Kotlin for the
Android app (Jetpack Compose UI). It reaches MVP alongside web and
server-sdk.

## Google login ceremony

`loginWithGoogle()` uses Google's native SDK (Credential Manager / Google
Identity Services) to obtain a Google-signed ID token directly, no browser
redirect (uauth-service/login-methods § Login sequences → Google (Android)).
This is what makes Google login "as low-friction as possible" on Android
(root HLD § Goals).

Credential Manager's underlying `getCredential()` call requires an Activity
context, not an application context, since it renders the picker as UI
anchored to that activity — an application-level context isn't sufficient.
`loginWithGoogle()` therefore takes the calling `Activity` as a parameter
(see Interface); this is Android-specific and not part of the
platform-agnostic client-sdk contract, since web's redirect-based ceremony
has no equivalent requirement.

Requesting an ID token from Credential Manager also requires passing the
consuming project's own `googleWebClientId` as `serverClientId` — the same
registered Google client ID used by web (uauth-service/login-methods §
Google client registration), which is what determines both the consent
screen the user sees (the app's own branding, not uauth's) and the `aud`
claim login-methods checks the resulting token against. This is app-level
configuration, not a `loginWithGoogle()` parameter (web § SDK configuration
covers the same concept; Android's mechanism for supplying it is not yet
decided, § Open Questions).

## Token storage

The refresh token (the session — uauth-service/sessions) must be stored
securely between app runs. Not yet decided: EncryptedSharedPreferences vs.
Android Keystore-backed storage directly (§ Open Questions).

## Login failure and recovery

Credential Manager returning no credential (no Google account on the device,
or the user cancels the picker) is a normal, expected outcome, not an
exception — `loginWithGoogle()` returns a "not logged in" result the calling
app can present its own retry/empty-state UI for.

If a stored refresh token survives an Android backup/restore or reinstall
onto a new device, but the corresponding session was revoked server-side in
the meantime (uauth-service/sessions), the resulting "session not found"
response (login-methods § Verification flow) is treated the same way: a
normal "not logged in" outcome that routes back to the login ceremony, not a
crash or fatal error.

## Interface

Kotlin signatures implementing the client-sdk contract:

```kotlin
suspend fun loginWithGoogle(activity: Activity): LoginResult
suspend fun logout()
fun getCurrentUser(): CurrentUser?
suspend fun getToken(): String?

sealed interface LoginResult {
    data class Success(val user: CurrentUser) : LoginResult
    object Failed : LoginResult
}

data class CurrentUser(
    val userId: String,
    val name: String? = null,
    val picture: String? = null,
    val email: String? = null,
)
```

`Failed` covers both a cancelled/missing Credential Manager result and a
server-side verification failure (§ Login failure and recovery).

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
3. How the app supplies its `googleWebClientId` for use as `serverClientId`
   (a build config value, a resource string, etc.) is not yet decided.

## References

- Parent: `docs/intent/client-sdk/client-sdk-design.md`
- Root HLD: `docs/high-level-design.md`
