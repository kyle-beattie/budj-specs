## 1. Unblock

- [x] 1.1 ~~Decide the Supabase client question (X1).~~ **Done.** Hand-rolled, no
      SDK — recorded as D16. The contract settled it: `/api/auth/sign-in`,
      `/api/auth/sign-up` and `/api/auth/refresh` are all server routes, leaving
      one direct call to `/auth/v1/token?grant_type=id_token`.
- [ ] 1.2 Agree the Akahu callback mechanism with `add-onboarding` (X2): what the
      server redirects to after the code exchange, and what the app registers to
      receive it. Blocks 12.2. **This is a gap in the server change too** — record
      the answer there as well.
- [ ] 1.3 Decide Google sign-in: GoogleSignIn SDK or Supabase OAuth through the
      web authentication session (X3). Blocks 10.5.
- [ ] 1.4 Decide whether a welcome screen precedes sign-in (X5). One screen and a
      copy decision; blocks nothing but 9.3.

## 2. Restructure and foundations

- [x] 2.1 ~~Create the folder tree from D1 and move the six existing files.~~
      **Done.** Confirmed by building: `Budj/` is a synchronised root group and
      picked up the new tree with no `project.pbxproj` edit.
- [x] 2.2 ~~Delete `ContentView.swift`.~~ **Done.** `Features/Placeholder/
      PlaceholderView.swift` replaces it, and `RootView` holds that instead.
- [x] 2.3 ~~Correct `CLAUDE.md`.~~ **Done.** It now records the synchronised
      groups, the isolation rule 2.5 turned up, the `BUDJ_API_BASE_URL` setting,
      and the no-dependency decision from D16.
- [x] 2.4 ~~Replace `CLAUDE.md`'s design-prototype section.~~ **Done.** The
      prototype and its generated design kit are out of date and are no longer
      cited anywhere: `CLAUDE.md` now points at the `budj-specs` store for
      behaviour and carries D4's visual rules directly. The stale rule model
      (a formula over `x`) went with it — `add-rule-allocation` replaced it with
      an ordered step list.
- [x] 2.5 ~~Confirm the default-MainActor settings behave as claimed.~~
      **Done, and it turned up something.** `-default-isolation MainActor` is
      genuinely applied, and it isolates *conformances* as well as types: an
      unannotated `struct` got a main-actor-isolated `Equatable`, warned about
      today and an error in the Swift 6 language mode. The wire types are
      therefore declared `nonisolated`. Note `SWIFT_VERSION` is still 5.0, so
      these are warnings rather than errors for now. The test target does not
      set the flag at all — test suites are annotated `@MainActor` instead.

## 3. Design system: tokens

- [x] 3.1 ~~`BudjColor`.~~ **Done.** Ten colour sets added, both appearances
      each. One extra the list did not anticipate: `onAccent`, dark in *both*
      appearances, because the accent is amber in both — a label that flips with
      the appearance is unreadable in one of them, which is exactly what the
      first render of the primary button looked like.
- [x] 3.2 ~~`BudjSpacing`.~~ **Done.** hair/tight/snug/regular/loose/section/screen.
- [x] 3.3 ~~`BudjRadius`.~~ **Done.** 12 / 20 / 28 / 32. "Full" is not a member:
      it is `Capsule`, which is a shape rather than a number, and buttons apply
      it directly.
- [x] 3.4 ~~`BudjTypography`.~~ **Done.** Plus a `badge` role, for the one place
      uppercase tracked text is permitted.
- [x] 3.5 ~~`BudjMotion`.~~ **Done.** Both curves, all three durations, and the
      Reduce Motion step transition.
- [ ] 3.6 Money formatting helper over `Decimal` with `FormatStyle`, NZD, two
      decimals, thousands separators, `.monospacedDigit()`. Not used much in
      onboarding — written here so the tabs inherit it rather than inventing it.

## 4. Design system: components

Each is its own file, takes values and closures only, and ships with previews for
every state.

- [x] 4.1 ~~`BudjButtonStyle`.~~ **Done.** Three variants, press scale 0.96,
      capsule, glow on primary alone, and an in-place loading state that keeps
      the button's size so the action area does not jump.
- [ ] 4.2 `BudjCard` — solid surface, 28pt radius, `@ViewBuilder` content.
- [ ] 4.3 `GlassSurface` — the signature treatment, mapped to iOS 26's native
      glass APIs rather than a hand-built blur. Reserved for surfaces floating
      over content.
- [x] 4.4 ~~`BudjTextField`.~~ **Done.** The error is part of the component, and
      the border changes with it, so the state is not carried by colour alone.
- [ ] 4.5 `BudjToggleRow` — label, optional footnote, binding.
- [ ] 4.6 `BudjBadge` — the one place uppercase tracked text is permitted.
- [ ] 4.7 `BudjBankRow` — institution name, idle/connecting/connected state, a
      button, distinguishable without colour.
- [x] 4.8 ~~`StepScaffold`.~~ **Done.** Title, subtitle, body and action slots;
      the action area is the floating surface, so it is where glass belongs.
      Capped at 560pt so onboarding stays a single column on iPad.
- [ ] 4.9 `PressScale` and `CardSurface` modifiers, exposed as `View` extensions.
      `PressScale` is done (it drops the scale but keeps the dimming under Reduce
      Motion); `CardSurface` waits for the first component that needs it.
- [ ] 4.10 `MoneyText` — a `Decimal` and a style, monospaced digits. Written now,
      used later.

## 5. Design system: gallery and accessibility

- [ ] 5.1 `ComponentGallery` rendering every component in every state, reachable
      in debug builds only.
- [ ] 5.2 Walk the gallery at the largest accessibility type size and fix
      everything that clips, truncates, or overlaps. This is the task that finds
      the problems; do not skip it because the default size looks fine.
- [ ] 5.3 Walk the gallery in both appearances. Light mode has never been looked
      at and every colour set defines it.
- [ ] 5.4 Verify every control has a text label under VoiceOver, including the
      visually icon-only ones.
- [ ] 5.5 Verify Reduce Motion degrades every animation to opacity, including the
      launch mark.

## 6. Core: networking

- [x] 6.1 ~~`ClientBuild`.~~ **Done.** Reads `CFBundleVersion` through an
      `InfoValues` seam, so a test can assert the real bundle's value parses —
      a `CFBundleVersion` of `1.0` would have every request refused.
- [x] 6.2 ~~`BudjAPI`.~~ **Done.** One request builder, so neither header has a
      per-call opportunity to be omitted. Includes the coalesced single-retry
      refresh: a network failure during refresh keeps the session, a refusal
      ends it. The base address comes from `BUDJ_API_BASE_URL`.
- [x] 6.3 ~~`APIError`.~~ **Done.** Codes are matched exactly, including case;
      a lowercase `subscription_required` degrades to a server failure rather
      than quietly sending someone to the paywall.
- [x] 6.4 ~~Central handling.~~ **Mechanism done** — `APIInterruption` and one
      `interruptionHandler` on the client, covered by tests. Wiring it to the
      screens it implies belongs to 8.3 and 9.x, which is where the screens are.
- [ ] 6.5 `Money` — integer-cents or `Decimal` backed, never `Double`.
- [ ] 6.6 Model types for the status, plan, and connection responses, decoded from
      the published contract's shapes. `OnboardingStatus` and `BudjSession` are
      done, along with the failure and collection envelopes; plan and connection
      remain.
- [ ] 6.7 Check the contract artifact's conformance vectors into the test target.

## 7. Core: session

- [x] 7.1 ~~`KeychainStore`.~~ **Done.** `WhenUnlockedThisDeviceOnly`, optional
      `.biometryCurrentSet`. Writes are delete-then-add rather than update,
      because an update cannot change the access control — turning biometry on
      for an existing session would otherwise silently do nothing.
- [x] 7.2 ~~`SessionStore`.~~ **Done.** `@Observable`, conforms to
      `SessionProviding` so the API client reads its token from the same object
      the root view routes on. Sign-out lives on `Authenticator`, which is what
      owns both the client and the store.
- [x] 7.3 ~~`BiometricGate`.~~ **Done.** `BiometricUnlock` has two cases and no
      error case: a cancelled prompt, a failed match, a changed enrolment and a
      device with no biometry at all are one answer — sign in again — and a
      caller that has to tell them apart is a caller that gets one of them
      wrong. Nothing enrolled short-circuits without prompting, so nobody is
      shown a prompt they cannot pass. `LAContext` is behind a
      `BiometricEvaluating` seam, and the real one builds a fresh context per
      evaluation because a reused one caches its success and can answer "yes"
      without asking anybody anything; `localizedFallbackTitle` is empty, since
      D8's fallback is sign-in rather than a second lock screen. It turned up a
      missing plist key: `NSFaceIDUsageDescription` was absent, which fails
      every evaluation on a Face ID device. The store side already existed:
      `SessionStore.restore()` turns a `KeychainError.notUnlocked` into no
      session plus `isLocked`.
- [x] 7.4 ~~Test: nothing sensitive is written to user defaults.~~ **Done.**
      Two assertions: no stored token appears anywhere in the defaults, and no
      app-owned key is named for a credential.

## 8. Launch

- [x] 8.1 ~~`LaunchGateModel`.~~ **Done.** Holds no state and renders nothing, so
      every outcome is reachable from a test. **"Unlock" turned out not to be a
      step of its own:** reading the Keychain item *is* the biometric prompt, so
      calling `BiometricGate.unlock()` first would raise a second one for the
      same unlock. The gate restores and lets `SessionStore` turn a cancelled
      prompt into no session. It also skips the read entirely when a session is
      already held, because the retry in 8.3 runs the same gate and would
      otherwise prompt again. Two mappings worth recording: a
      `subscription_required` refusal is `billing` rather than unreachable — the
      server naming the reason is the server naming the step — while a 500, a
      decode failure and an offline device are one answer, since they differ in
      cause and not in what the app can do.
- [x] 8.2 ~~`RootView`.~~ **Done.** The 600ms floor was already there; it now
      holds over `AppModel.start()` rather than over nothing.
- [x] 8.3 ~~The launch outcomes, each with a screen.~~ **Done.** `AppPhase` grew
      `unreachable` and `onboarding` now carries the server's step, so the
      resolved destination survives into the router 9.2 will build.
      `UnreachableView` is the one that would have been skipped: it says it
      cannot reach Budj and that nothing is wrong with the user's account, which
      at that point the app does not know to be false. Its retry re-runs the
      gate rather than a second code path. Steps land on
      `OnboardingStepPlaceholder`, which names the step rather than pretending
      to be it, until 9.3/11.x/12.x.
- [x] 8.4 ~~`MustUpdateView`.~~ **Done.** No dismissal, and it says the app must
      be updated rather than that Budj is unavailable. The App Store URL is a
      placeholder until there is an app id.
- [x] 8.5 ~~Unit tests over the gate.~~ **Done.** Ten, covering all four
      outcomes, the unsupported-client override, a refused session being cleared
      rather than retried, an unreadable Keychain item, and the retry not
      re-reading the Keychain. The three "cannot be asked" causes are asserted
      separately so a future change that starts guessing a step from one of them
      fails a test.

## 9. Onboarding router

- [ ] 9.1 `OnboardingStep` enum and `OnboardingModel` — holds the server's step,
      the advisory push flag, and the local record of which client-only steps have
      been offered.
- [ ] 9.2 `OnboardingFlowView` — switches over the step with a value-driven
      transition. Not a `NavigationStack`; see D13.
- [ ] 9.3 The screens: welcome (if 1.4 says so), sign-in, paywall, biometric
      opt-in, connect bank, push opt-in, each on `StepScaffold`.
- [ ] 9.4 Refresh status after purchase submission and after the bank session
      returns, and after nothing else.
- [ ] 9.5 Unit tests: the model never advances on a local success alone; a
      `subscription_required` failure returns it to billing; a client-only step
      already offered is not offered again.

## 10. Identity

- [ ] 10.1 `SignInWithAppleButton`, requesting full name and email.
- [ ] 10.2 Send the credential's **identity token** to Supabase.
- [ ] 10.3 Send the credential's **authorization code** to the Budj server, after
      the session exists.
- [ ] 10.4 Test asserting 10.2 and 10.3 send the *right artifact to the right
      place*. This is the single highest-value test in the change: getting it
      wrong produces a working app and an undeletable account, discovered in
      `add-account-deletion` months later.
- [ ] 10.5 Google sign-in by whichever route 1.3 chose.
- [x] 10.6 ~~Email and password sign-in and registration.~~ **Done.** One screen
      for both. Registration that returns no session is its own outcome rather
      than a failure — that is what a project with email confirmation switched on
      does, and an app that guesses shows an error to someone whose account was
      created perfectly. A refused sign-in says neither which field was wrong nor
      that a session ended: it announced `sessionEnded` at first, which would
      have bounced people off the screen they were signing in on.
- [ ] 10.7 A failed authorization-code exchange is logged and does not fail
      sign-in.
- [ ] 10.8 Forward Apple's name components when present; never derive a name from
      an email address. Test the private-relay case explicitly.
- [x] 10.9 ~~Sign-out clears the Keychain item and returns to the entry point.~~
      **Done.** The server call is best-effort: someone signing out on a train has
      still signed out, so the local session is cleared whether the request
      succeeded, failed, or was refused.

## 11. Subscription

- [ ] 11.1 Load products with StoreKit 2 and render `displayPrice` only. Grep the
      paywall for a currency symbol afterwards; there should be none.
- [ ] 11.2 Purchase, verify locally, submit the signed transaction, refresh status.
- [ ] 11.3 Finish the transaction only after submission succeeds, so a failure is
      re-observed rather than lost.
- [ ] 11.4 Observe `Transaction.updates` from app start, not from the paywall.
- [ ] 11.5 Restore, via App Store sync and resubmission.
- [ ] 11.6 Distinct presentation for a transaction already bound to another
      account.
- [ ] 11.7 No cancellation control anywhere; direct to App Store subscription
      management instead.
- [ ] 11.8 StoreKit configuration file for local testing, so the paywall can be
      exercised without App Store Connect.

## 12. Bank connection

- [ ] 12.1 Request the authorisation URL from the server and present it with
      `@Environment(\.webAuthenticationSession)`, ephemeral session off.
- [ ] 12.2 Handle the callback agreed in 1.2 — and treat completion, cancellation,
      and dismissal identically: re-fetch status and render the server's answer.
- [ ] 12.3 `ConnectBankView` making no assumption that it is the first connection,
      so settings can reuse it.
- [ ] 12.4 Inline handling of `plan_limit_exceeded` without leaving the step.
- [ ] 12.5 Copy stating read-only access and revocability, and implying no ability
      to move money.
- [ ] 12.6 Test: cancelling the session leaves the user on the bank step with no
      error that suggests they did something wrong.

## 13. Push

- [ ] 13.1 Push opt-in screen after the bank step, with a skip of equal
      prominence.
- [ ] 13.2 Request authorisation, register for remote notifications, and forward
      the APNs token to the server.
- [ ] 13.3 `UIApplicationDelegateAdaptor` whose only job is receiving the token.
      Keep it to a handful of lines; it is the one unavoidable UIKit touchpoint.
- [ ] 13.4 Record that the offer was made and never make it again.
- [ ] 13.5 Test: declining reaches `ready`.

## 14. Tests

- [ ] 14.1 Swift Testing over the launch gate, the onboarding model, the error
      mapping, and money decoding — all pure, all cheap, all exhaustible.
- [ ] 14.2 Assert the published conformance vectors against the app's own decoder.
- [ ] 14.3 XCUITest: first launch through to `ready`.
- [ ] 14.4 XCUITest: abandon at the bank step, relaunch, land on the bank step.
      The most likely regression and the least likely thing anyone checks by hand.
- [ ] 14.5 XCUITest: the unsupported-client response produces the update prompt
      with no way back.

## 15. Before this is considered done

- [ ] 15.1 Configure the required capabilities: Sign in with Apple, In-App
      Purchase, push notifications, and an associated domain if 1.2 chose one.
- [ ] 15.2 Confirm nothing in the repository holds a secret — the app has no
      server credentials and should acquire none.
- [ ] 15.3 Read the flow's copy against the voice rules in D4: sentence case,
      second person, no emoji, no exclamation points.
- [ ] 15.4 Run the app on an iPad simulator. `TARGETED_DEVICE_FAMILY` is `"1,2"`;
      onboarding stays single-column but must not look broken.
