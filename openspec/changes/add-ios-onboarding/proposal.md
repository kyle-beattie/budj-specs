## Why

`add-onboarding` specifies a server that can take a person from "downloaded the
app" to "paying customer with a connected New Zealand bank". It specifies none of
the app that does the taking. Every step of that path is a screen — sign in with
Apple, buy a subscription, hand off to a bank, come back — and the app owns the
half of each step the server never sees.

The gap is not cosmetic. The server's contract quietly assigns the client work
that is easy to get wrong and expensive to discover late:

- The Apple **identity token** goes straight to Supabase and the Apple
  **authorization code** comes here, and they are different artifacts (D2, D13).
  Capture the wrong one at sign-in and account deletion is impossible forever.
- Onboarding step is **derived, never stored** (D1). The app must treat
  `GET /api/onboarding/status` as the only authority on where a person is, rather
  than remembering locally where it left them.
- Face ID is **entirely a client concern** (D10). The server has no endpoint, no
  column, and no opinion. If the app does not build it, it does not exist.
- Every request must carry a **build identifier**, and a request without one is
  treated as unsupported rather than exempt. An app that forgets it is an app
  that cannot start.

Separately, this is the change that decides how the app is *organised*. There are
four more feature areas behind onboarding — rules, accounts, approvals, settings —
and the structure laid down here is the one they will all be built into. Getting a
reusable component layer in place while the surface is six screens is cheap;
retrofitting one across thirty is not.

## What Changes

- **A launch gate that resolves session before showing anything.** The existing
  fixed 1.2-second `RootView` hold becomes a real gate: restore the session from
  the Keychain, unlock it with Face ID if enrolled, fetch onboarding status, and
  route. The launch screen stays up while that happens rather than for a
  hard-coded interval.
- **The onboarding flow is a router over a server-derived step**, not a wizard
  the app walks forward through. Status is re-fetched after every step that
  changes server state; the app never advances itself on optimism.
- **Native sign-in with Apple and Google, plus email and password.** Identity
  token to Supabase directly; Apple's authorization code posted to this server;
  a failed code exchange never fails sign-in.
- **A paywall, because there is no free tier.** StoreKit 2 purchase, signed
  transaction submitted to the server, and — critically — the app never formats a
  price itself; `Product.displayPrice` is the only price shown.
- **Face ID opt-in, implemented locally.** The Supabase refresh token is held in
  the Keychain behind a biometric access control. Declining is a supported
  choice, not a degraded one, and a biometric failure falls back to sign-in
  rather than locking the user out.
- **Bank connection by hand-off.** The server builds the Akahu authorisation URL;
  the app presents it in an `ASWebAuthenticationSession` and, on return, re-asks
  the server what happened. The app never sees an Akahu token and never calls
  Akahu.
- **Push opt-in as an advisory step that can be skipped**, matching D12 — offered
  after the bank connects, where the value of a notification is obvious, and
  never blocking `ready`.
- **A design system layer with a component gallery.** Tokens, primitives, and a
  preview gallery that renders every component in every state, so the visual
  language is reviewable without navigating the app to find it.
- **A typed API client that fails in ways the UI can act on.** `402
  subscription_required`, `plan_limit_exceeded`, and the unsupported-client code
  are distinguishable outcomes, not a generic error banner.
- **A folder structure organised by feature, with one type per file.** Xcode's
  file-system-synchronised groups mean the layout on disk *is* the layout in the
  project, so this is free to do now and awkward later.

Explicitly out of scope: the four main tabs (home, rules, accounts, settings),
the rule builder, approval review, and account deletion. This change ends the
moment status reports `ready` and hands off to a placeholder.

## Capabilities

### New Capabilities

- `app-launch`: Cold start, launch-screen handover, session restoration,
  biometric unlock, and the routing decision that follows.
- `app-onboarding`: The step router, resumability across launches, the order and
  skippability of each step, and recovery when status cannot be fetched.
- `app-identity`: Apple, Google and password sign-in surfaces; the identity-token
  and authorization-code split; session custody; sign-out.
- `app-subscription`: Plan presentation, StoreKit purchase and restore,
  transaction submission, and what the app does when entitlement is lost while it
  is running.
- `app-bank-connect`: Presenting the Akahu authorisation session, returning from
  it, confirming the outcome with the server, and connecting more than one bank.
- `app-api-client`: The build identifier on every request, authorisation, typed
  error codes, and conformance to the published contract artifact.
- `app-design-system`: Tokens, component primitives, composition rules, the
  preview gallery, and the accessibility floor every component must clear.

### Modified Capabilities

None. This is the first change describing the iOS client; the server
capabilities in `add-onboarding` are consumed here, not amended.

## Impact

**Repository.** `budj` (the iOS app), which today contains six Swift files: an
app entry point, a placeholder `ContentView`, a launch screen, and the brand
mark. Everything else is new.

**Depends on `add-onboarding` for its contract, not its completion.** The app can
be built against the published OpenAPI artifact and its conformance vectors
before the server implements a single route, and should be — the contract exists
precisely because the two repositories cannot see each other's source (D14).
Nothing here can be *verified* end to end until the server ships.

**Structure.** The existing flat `Budj/` folder is reorganised into `App/`,
`DesignSystem/`, `Features/`, and `Core/`. `Budj/` is a
`PBXFileSystemSynchronizedRootGroup`, so files and folders added on disk join the
target automatically — the note in `CLAUDE.md` that new sources must be
registered in `project.pbxproj` is out of date and is corrected by this change.

**Design source of truth moves into the repository.** The asset catalogue holds
the colour — a warm off-white and near-black background pair with both
appearances, and a cyan/amber/green mark — and D4 holds the rest: the spacing
scale, the full-radius system, glass usage, motion, type, and voice. There is no
external design document, so the component gallery is what stops the language
drifting.

**Capabilities and entitlements.** Sign in with Apple, In-App Purchase, push
notifications, Keychain sharing for the biometric-protected item, and an
associated domain if the Akahu return uses a universal link. Several require App
Store Connect configuration that cannot be done from the repository.

**External dependencies.** None intended. `AuthenticationServices`, `StoreKit`,
`LocalAuthentication`, and `UserNotifications` are system frameworks. The one
open decision is the Supabase Swift client, which is third party and which the
project's rules require asking about before adding — X1.

**Testing.** Unit tests with Swift Testing over the step router, the money and
status decoders, and the error mapping — all pure and cheap to test exhaustively.
XCUITest for the flow itself, including the resume-after-abandonment path, which
is the one most likely to regress and the least likely to be exercised by hand.

**Risk.** Two failures here are silent. Capturing the wrong Apple artifact at
sign-in produces a working app and an account that can never be properly deleted,
discovered months later in a different change. Omitting the build identifier
produces an app that works against a permissive server and is refused the day the
gate is switched on. Both are cheap to test for and impossible to notice by
using the app.
