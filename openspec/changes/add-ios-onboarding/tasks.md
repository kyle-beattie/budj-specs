## 1. Unblock

- [ ] 1.1 Decide the Supabase client question (X1): adopt `supabase-swift` or
      hand-roll the GoTrue calls. Blocks section 10 only. Record the answer in
      `design.md` either way — "we chose not to" is worth as much as "we chose to".
- [ ] 1.2 Agree the Akahu callback mechanism with `add-onboarding` (X2): what the
      server redirects to after the code exchange, and what the app registers to
      receive it. Blocks 12.2. **This is a gap in the server change too** — record
      the answer there as well.
- [ ] 1.3 Decide Google sign-in: GoogleSignIn SDK or Supabase OAuth through the
      web authentication session (X3). Blocks 10.5.
- [ ] 1.4 Decide whether a welcome screen precedes sign-in (X5). One screen and a
      copy decision; blocks nothing but 9.3.

## 2. Restructure and foundations

- [ ] 2.1 Create the folder tree from D1 and move the six existing files into it.
      No `project.pbxproj` edit is needed — `Budj/` is a synchronised root group.
      Build to confirm that is actually true before relying on it further.
- [ ] 2.2 Delete `ContentView.swift`. It is a placeholder whose name means
      nothing; `Features/Placeholder/` replaces it.
- [ ] 2.3 Correct `CLAUDE.md`: new sources are picked up automatically, not
      registered by hand.
- [x] 2.4 ~~Replace `CLAUDE.md`'s design-prototype section.~~ **Done.** The
      prototype and its generated design kit are out of date and are no longer
      cited anywhere: `CLAUDE.md` now points at the `budj-specs` store for
      behaviour and carries D4's visual rules directly. The stale rule model
      (a formula over `x`) went with it — `add-rule-allocation` replaced it with
      an ordered step list.
- [ ] 2.5 Confirm `SWIFT_VERSION` and the default-MainActor settings behave as
      `CLAUDE.md` claims, by writing one unannotated `@Observable` class and
      seeing it compile without `@MainActor`.

## 3. Design system: tokens

- [ ] 3.1 `BudjColor`: semantic names resolving asset catalogue colours —
      background, surface, raised, text primary/secondary/tertiary, accent,
      danger, warning, border subtle/strong. Add the missing colour sets to the
      catalogue with both appearances defined.
- [ ] 3.2 `BudjSpacing`: a 4pt scale, named rather than numeric.
- [ ] 3.3 `BudjRadius`: 12 / 20 / 28 / 32 / full. Nothing in the app uses a radius
      not on this list.
- [ ] 3.4 `BudjTypography`: display, headline, title, body, caption, and label
      roles mapped onto Dynamic Type text styles. No fixed point sizes.
- [ ] 3.5 `BudjMotion`: a standard curve, a springier curve for toggles and sheet
      presentation, and three durations — 120 / 200 / 340ms.
- [ ] 3.6 Money formatting helper over `Decimal` with `FormatStyle`, NZD, two
      decimals, thousands separators, `.monospacedDigit()`. Not used much in
      onboarding — written here so the tabs inherit it rather than inventing it.

## 4. Design system: components

Each is its own file, takes values and closures only, and ships with previews for
every state.

- [ ] 4.1 `BudjButtonStyle` — primary, secondary, and quiet variants; press scale
      0.96; full radius; the accent glow reserved for primary.
- [ ] 4.2 `BudjCard` — solid surface, 28pt radius, `@ViewBuilder` content.
- [ ] 4.3 `GlassSurface` — the signature treatment, mapped to iOS 26's native
      glass APIs rather than a hand-built blur. Reserved for surfaces floating
      over content.
- [ ] 4.4 `BudjTextField` — label, placeholder, error text, secure variant.
- [ ] 4.5 `BudjToggleRow` — label, optional footnote, binding.
- [ ] 4.6 `BudjBadge` — the one place uppercase tracked text is permitted.
- [ ] 4.7 `BudjBankRow` — institution name, idle/connecting/connected state, a
      button, distinguishable without colour.
- [ ] 4.8 `StepScaffold` — title, `@ViewBuilder` body, primary action, optional
      secondary action. Every onboarding screen is built on it.
- [ ] 4.9 `PressScale` and `CardSurface` modifiers, exposed as `View` extensions.
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

- [ ] 6.1 `ClientBuild` — reads the bundle build number; one value, no hand-written
      constant.
- [ ] 6.2 `BudjAPI` — one type, applies authorization and the build identifier to
      every request. There must be no way to construct a request without them.
- [ ] 6.3 `APIError` — the enum from D12, decoding the server's error codes into
      distinct cases and degrading unknown codes to a general server failure.
- [ ] 6.4 Central handling for `unsupportedClient`, `subscriptionRequired`, and
      `unauthorized`, applied once rather than at each call site.
- [ ] 6.5 `Money` — integer-cents or `Decimal` backed, never `Double`.
- [ ] 6.6 Model types for the status, plan, and connection responses, decoded from
      the published contract's shapes.
- [ ] 6.7 Check the contract artifact's conformance vectors into the test target.

## 7. Core: session

- [ ] 7.1 `KeychainStore` — device-only accessibility, optional biometric access
      control using `.biometryCurrentSet`.
- [ ] 7.2 `SessionStore` — an `@Observable` holding the current session, reading
      and writing through the Keychain, exposing sign-out.
- [ ] 7.3 `BiometricGate` — availability check and unlock; a cancelled or failed
      prompt returns a definite "fall back to sign-in", not an error the caller
      has to interpret.
- [ ] 7.4 Test: nothing sensitive is written to user defaults. Assert it rather
      than believing it.

## 8. Launch

- [ ] 8.1 `LaunchGateModel` — restore, unlock, fetch status, resolve a destination.
      Pure enough to unit test with a stubbed API.
- [ ] 8.2 Rewrite `RootView` to hold the launch screen over that work, with a
      minimum duration floor, replacing the fixed `Task.sleep`.
- [ ] 8.3 The four launch outcomes from D6, each with a screen: sign-in, a step,
      ready, and a retryable connection failure. The failure case is the one that
      will be skipped; do not skip it.
- [ ] 8.4 `MustUpdateView` — terminal, no way back, distinguishable from an outage.
- [ ] 8.5 Unit tests over the gate covering all four outcomes plus the
      unsupported-client override.

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
- [ ] 10.6 Email and password sign-in and registration, as the fallback.
- [ ] 10.7 A failed authorization-code exchange is logged and does not fail
      sign-in.
- [ ] 10.8 Forward Apple's name components when present; never derive a name from
      an email address. Test the private-relay case explicitly.
- [ ] 10.9 Sign-out clears the Keychain item and returns to the entry point.

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
