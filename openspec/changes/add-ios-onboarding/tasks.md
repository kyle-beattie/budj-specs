## 1. Unblock

- [x] 1.1 ~~Decide the Supabase client question (X1).~~ **Done.** Hand-rolled, no
      SDK — recorded as D16. The contract settled it: `/api/auth/sign-in`,
      `/api/auth/sign-up` and `/api/auth/refresh` are all server routes, leaving
      one direct call to `/auth/v1/token?grant_type=id_token`.
- [ ] 1.2 Agree the Akahu callback mechanism with `add-onboarding` (X2): what the
      server redirects to after the code exchange, and what the app registers to
      receive it. Blocks 12.2. **This is a gap in the server change too** — record
      the answer there as well.
- [x] 1.3 ~~Decide Google sign-in (X3).~~ **Deferred, deliberately.** Neither
      route is chosen and Google is not in this change. Apple plus email
      satisfies review, and picking between a third-party SDK and a browser
      round trip under onboarding's deadline is how the zero-dependency
      position gets spent by accident. Revisit once onboarding ships; the
      Supabase-OAuth route will be cheaper then, because task 12 builds the
      `ASWebAuthenticationSession` machinery it needs.
- [x] 1.4 ~~Decide whether a welcome screen precedes sign-in (X5).~~ **Done —
      yes.** `WelcomeView` is the first screen, with "Create an account" and "I
      already have an account" as two actions of equal weight; both open the
      same screen and differ only in which mode it starts in. It is the entry
      point rather than a step completed once, so signing out returns you to it.
      Recorded as X5 in the design.

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

- [x] 9.1 ~~`OnboardingStep` and `OnboardingModel`.~~ **Done, and D5 needed
      resolving.** D5 says Face ID is offered "immediately after sign-in" and
      also that "the paywall sits between sign-in and Face ID". Those cannot
      both hold: after sign-in the server says `billing`, so Face ID cannot
      precede it. **Resolved against D5's paywall line rather than for it:**
      the order is welcome → signIn → biometrics → billing → bank → push →
      ready. "Billing before bank" is a statement about the *server's* step
      order, and Face ID is not one of the server's steps, so nothing forced
      the paywall in front of it — and meeting the price as the first thing
      after signing in is worse for no gain. Face ID is keyed off the first
      server step after signing in, whatever it is, rather than off `billing`:
      keying it to `billing` would skip the offer entirely for somebody who
      reinstalls, signs in and lands on `bank` because they already subscribed.
      Push is offered before `ready`. D5 has been updated. There is deliberately no `advance()` — the step moves
      on a refresh or on a client-only step being recorded, and by no other
      means. `ClientStepRecord` is a seam over user defaults; the real keys are
      named for the step and never for what it turns on, so 7.4's assertion
      about credential-shaped key names keeps holding.
- [x] 9.2 ~~`OnboardingFlowView`.~~ **Done.** A switch with
      `BudjMotion.stepTransition(reduceMotion:)`, animated on the step value.
      `AppPhase` lost the `resuming` payload it gained in 8.3: the router holds
      the step now, and two objects holding it is two objects that can
      disagree. `AppModel` owns the router for the same reason — it is what
      turns a launch destination into a step.
- [x] 9.3 ~~The screens: welcome, sign-in, paywall, biometric opt-in, connect
      bank, push opt-in, each on `StepScaffold`.~~ **Done — all six.**
      Sign-in landed in 10.6. `WelcomeView` is new here now 1.4 has answered:
      what Budj does, then two actions of equal weight. `BiometricOptInView` is
      new here too, because 7.3 and 7.2 gave it everything it needs — it names
      the biometry, turning it on rewrites the stored session behind the access
      control, and both answers are recorded identically because the router
      only cares that it asked.
      **The last three arrived with their behaviour deliberately incomplete**,
      because the server derives `billing` and `bank` from a real subscription
      row and a real Akahu token (`onboarding.service.ts`) and neither exists
      yet. So `PaywallView` and `ConnectBankView` are real screens with their
      real copy, and their primary actions are **present and disabled** rather
      than wired to something that looks like buying or connecting. Past them is
      `DebugStepSkip` — a separate, labelled control, the whole file inside
      `#if DEBUG`, so a release build has no way through except the real one. A
      "Subscribe" button that silently advanced would be a lie that outlives the
      reason for it. `PushOptInView` is not mocked at all: push is client-only,
      so there was nothing to stand in for.
      `OnboardingStepPlaceholder` is deleted — every step now has a screen.
- [x] 9.4 ~~Refresh after the steps that change server state, and after nothing
      else.~~ **Mechanism done** — `OnboardingModel.refresh()`, with a
      `subscription_required` refusal mapping to billing plus an explanation.
      The two call sites are 11.2 and 12.2, which is where those steps are. A
      failed refresh is explicitly not progress: the step on screen stays put
      rather than being replaced by a guess.
- [x] 9.5 ~~Unit tests.~~ **Done.** Seventeen, covering all three required
      cases plus the whole order in one pass, Face ID offered before whichever
      server step comes first, a client step answered without a round trip,
      declining recorded the same as accepting, signing out returning to
      welcome, and a device with no biometry not being offered it — and not
      recorded as offered either, so a device that gains an enrolment can still
      be asked.

- [x] 9.6 ~~Fix biometric unlock, which did not survive contact with a device.~~
      **Done.** Three defects, one per symptom reported, each now with a test
      that has been seen to fail against the old code (D8a):
      **(a)** `SessionStore.requiresBiometry` started `false` every launch and
      every write consults it, so the first token refresh after relaunching
      rewrote the session without its access control — biometry silently gone,
      permanently. It is now read at construction from `BiometricPreferenceStore`.
      **(b)** "Use Face ID" never prompted, because `SecItemAdd` does not raise a
      prompt — only reading does. The step now evaluates the biometric first, and
      a refused write is reported rather than swallowed by a `try?`.
      **(c)** The offered-record was keyed per install, so a second account on the
      same device was never offered biometry. `ClientOnlyStep` now carries a
      scope: push stays device-wide because iOS will not re-prompt, biometrics is
      per user because a new session is a new question.
      Verified in the simulator as well — the system Face ID prompt now appears
      on the call the button makes, which is the part no unit test can assert.

## 10. Identity

- [x] 10.1 ~~`SignInWithAppleButton`.~~ **Done.** `.fullName` is requested even
      though Apple supplies it only on the first authorisation: it is the single
      opportunity to seed a display name, and not asking means never having it.
      Apple's own button rather than a `BudjButtonStyle` one — its appearance is
      prescribed by the HIG and re-skinning it is a review rejection, so it is
      the one control in the app that does not come from the design system.
      Cancelling is silent; it is a sheet the person closed, not a failure.
- [x] 10.2 ~~Identity token to Supabase.~~ **Done**, via `SupabaseIdentity` —
      the one direct call, in `Core/Session` rather than `Core/Networking`
      because it carries no bearer token and no `X-Client-Build` (D16).
      **It needed its own coders.** GoTrue's shape is not the contract's:
      snake_case keys, a Unix `expires_at`, and `email_confirmed_at` as a
      timestamp where the contract has a boolean — and it emits fractional
      seconds, which `.iso8601` refuses outright. Sharing `BudjAPI`'s decoder
      would have failed at runtime on the first real sign-in. The translation to
      `BudjSession` happens once, so nothing downstream can tell how the user
      got here. A Supabase refusal is mapped to a distinct code rather than to
      `unauthorized`, which would have announced that a session ended when the
      truth is that one was never established.
- [x] 10.3 ~~Authorization code to the Budj server.~~ **Done**, after the
      session exists, because that route is authenticated. Asserted by ordering
      rather than by inspection: the test checks the grant request carries the
      bearer token Supabase had just issued.
- [x] 10.4 ~~The right artifact to the right place.~~ **Done, and verified by
      mutation.** Three independent assertions: each artifact reaches its own
      destination, no identity token appears in any body sent to the Budj
      server, and no authorization code appears in any body sent to Supabase.
      The two negative assertions exist because the positive one alone would
      still pass if a fixture happened to use the same value for both. The
      artifacts were then deliberately swapped in `Authenticator` to confirm the
      suite fails — all three caught it. A test for a defect with no symptom is
      worth nothing unless it has been seen to fail.
- [x] 10.5 ~~Google sign-in.~~ **Out of scope — 1.3 deferred it.** The app
      offers Apple and email. `SupabaseIdentity.Provider` keeps its `google`
      case and the exchange is provider-agnostic, so whichever route is chosen
      later plugs into the same call; nothing here needs undoing first.
- [x] 10.6 ~~Email and password sign-in and registration.~~ **Done.** One screen
      for both. Registration that returns no session is its own outcome rather
      than a failure — that is what a project with email confirmation switched on
      does, and an app that guesses shows an error to someone whose account was
      created perfectly. A refused sign-in says neither which field was wrong nor
      that a session ended: it announced `sessionEnded` at first, which would
      have bounced people off the screen they were signing in on.
- [x] 10.7 ~~A failed exchange does not fail sign-in.~~ **Done.** Two ways to
      fail and both are survivable: the route answering non-200, and it
      answering 200 with `stored: false` because Apple refused. Neither undoes
      the session. The code is single-use, so there is nothing to retry, and
      refusing somebody entry over a deletion concern they will not meet for
      months is the wrong trade.
- [x] 10.8 ~~Names.~~ **Done in the app, and it turned up a gap that is not the
      app's to close.** `AppleCredential` has no path from an email address to a
      name; blank components are `nil` rather than an empty string; the
      private-relay case is asserted by checking the relay address appears
      nowhere in the request body at all, rather than by checking one field.
      **The gap:** the server's `handle_new_user` trigger reads `full_name` from
      `raw_user_meta_data`, and GoTrue populates that from the *id-token
      claims* — but Apple does not put the name in the JWT. It arrives beside
      it, once. D16 says to send `data.full_name` on the id-token exchange, and
      the app does, but GoTrue's `id_token` grant has no documented `data`
      parameter and very likely ignores it. If so the display name is never
      seeded for Apple sign-in, silently, and the migration's own comment says
      it is then unrecoverable. Needs deciding with `add-onboarding`: either a
      server route that accepts the name alongside the Apple grant, or the app
      calling `PATCH /auth/v1/user` — though the trigger fires on insert, so
      that alone would not seed `profiles`.
- [x] 10.9 ~~Sign-out clears the Keychain item and returns to the entry point.~~
      **Done.** The server call is best-effort: someone signing out on a train has
      still signed out, so the local session is cleared whether the request
      succeeded, failed, or was refused.

- [x] 10.10 ~~Agree the confirmation redirect with `add-onboarding` (D17).~~
      **Done, and built** — see `add-onboarding` 4.13. **Not
      the app scheme directly:** `emailRedirectTo` is an `https://` page on our
      own server, because mail clients open links in embedded browsers that will
      not follow a redirect into a custom scheme, and some pre-fetch links and
      burn the single-use token before anybody taps. So `add-onboarding` serves
      `{PUBLIC_URL}/auth/confirm`, points `AUTH_REDIRECT_URL` at it, and adds it
      to Supabase's redirect allow list. That route must be **a page rather than
      a 302** — the session arrives in the URL fragment, which is never sent to
      the server, so only the browser can read it and hand it to
      `budj://auth/confirm`. A visible "Open Budj" button for the case where the
      automatic hop is blocked. Nothing in the app changed for it; the URL scheme
      was already registered and the parse already read both halves of the URL.
      **One thing the end-to-end run added:** the redirect carries `type=signup`,
      and a recovery link would carry `type=recovery` on the same shape and with
      a session of its own. The server keeps the two redirects apart, but the
      parse now refuses `recovery` outright rather than leaving that as the only
      lock.
- [x] 10.11 ~~`EmailConfirmationLink` — read the redirect.~~ **Done.** Values
      come from the fragment as well as the query, and both are read from the
      *percent-encoded* strings: `URLComponents.fragment` hands back a decoded
      value, which mangles the token it is there to carry. Decoding is form
      decoding rather than URL decoding, because Supabase writes the spaces in
      `error_description` as `+`. A confirmation URL carrying nothing usable is
      answered as a refusal rather than ignored — the person is standing in front
      of the app having just tapped the link, and is owed an answer. Everything
      on another host or path answers nothing at all, which is what lets one
      `onOpenURL` serve the bank callback too.
- [x] 10.12 ~~`EmailConfirmationModel` and `ConfirmEmailSheet`.~~ **Done.** One
      sheet, four states, and `ConfirmEmailNotice` is gone — it was an inline
      notice on the one screen guaranteed not to be there when the answer
      arrives. The model is built in `BudjApp` beside the authenticator and the
      sheet is presented from `RootView`, so neither depends on the sign-in
      screen having ever been shown. The exchange state carries no button and
      disables interactive dismissal: there is nothing useful to press, and a
      swipe landing between the link and the session would leave the app looking
      signed out to someone who is not.
- [x] 10.13 ~~Exchange the link's refresh token for a session.~~ **Done**, via
      `Authenticator.completeEmailConfirmation(refreshToken:)` over a new
      unauthenticated `Endpoint.exchange(refreshToken:)`. The *refresh* token is
      what is spent, not the access token beside it in the fragment: the fragment
      carries no user and `BudjSession` has one, and going through
      `/api/auth/refresh` means the session arrives in the same shape as every
      other one. Closing the sheet after a success runs `AppModel.signedIn()`,
      which re-runs the gate — the step is the server's answer, not a guess.
      Closing by swipe goes through the same path as the button, so a confirmed
      session is not stranded behind a gesture.
- [x] 10.14 ~~Tests.~~ **Done.** Twelve in `EmailConfirmationTests`, including
      the two that matter: the fragment is what is read, and the value posted to
      `/api/auth/refresh` is the refresh token, with the link's access token
      appearing nowhere in the body. An expired link is asserted to reach no
      server at all. Verified in the simulator as well — iOS resolves `budj://`
      to the app, and all three visible states render.

## 11. Subscription

> **Paused after 11.1 and 11.8, deliberately.** Reaching the trades needs an
> Android app, and two app stores means two in-app purchase systems — StoreKit
> and Google Play Billing — each with its own receipt verification, its own
> notification webhook, and its own handling of refunds, upgrades and grace
> periods, plus entitlement reconciliation between them. One web checkout serves
> both platforms for roughly 3% instead of 15–30%. On ~2,000 subscribers at
> $199/year that difference is six figures, and it is a decision worth making
> *before* the purchase path exists rather than after. 11.2–11.7 stay unbuilt
> until the rail is chosen; the debug skip on the paywall means nothing is
> blocked by that. 11.1 and 11.8 are kept either way — they cost nothing to
> hold, and if App Store billing wins they are already done.


- [x] 11.1 ~~Load products with StoreKit 2 and render `displayPrice` only.~~
      **Done, and it is real** — the plans, prices and cadence on the paywall
      come from `Product.products(for:)`, not from a fixture. All contact with
      `Product` is in `StoreKitPlanOffers`, which reduces it to `PlanOffer`:
      a type carrying `displayPrice` as a **string** and no numeric price and no
      currency code at all. Nothing downstream is given a number it could format,
      which is a stronger guarantee than remembering not to. Grepped, as
      instructed: no currency symbol in the paywall or the row.
      The seam (`PlanOfferLoading`) exists because `SKTestSession` needs a
      running StoreKit environment and `PaywallModel` is pure — five tests over
      it, including the one the screen rests on: a store that cannot be asked
      leaves the *price* unknown rather than the screen failed, because the plan
      exists whether or not the App Store is reachable.
      **One plan now, not two** (see the pricing note below), so `PaywallModel`
      has no selection state and `PlanOptionRow` became `PlanSummaryCard` — with
      a single plan, presenting it as a choice is theatre.
- [ ] 11.2 Purchase, verify locally, submit the signed transaction, refresh status.
- [ ] 11.3 Finish the transaction only after submission succeeds, so a failure is
      re-observed rather than lost.
- [ ] 11.4 Observe `Transaction.updates` from app start, not from the paywall.
- [ ] 11.5 Restore, via App Store sync and resubmission.
- [ ] 11.6 Distinct presentation for a transaction already bound to another
      account.
- [ ] 11.7 No cancellation control anywhere; direct to App Store subscription
      management instead.
- [ ] 11.9 **Collapse the server's plan catalogue to one plan.** `plans.ts` still
      has `starter` and `pro`; the app now knows only `com.budj.standard.yearly`,
      so a real submission would not resolve. Nine test files reference the two
      codes, and `bank-connections.service.ts` gates on `plan.maxConnections`, so
      the limits should stay as abuse guardrails rather than being deleted — they
      are simply no longer a product tier and are not marketed. Belongs to
      `add-onboarding`, not here, but it is this change that invalidated it.
- [x] 11.8 ~~StoreKit configuration file for local testing.~~ **Done.**
      `Budj/Budj.storekit`, referenced from the `Budj` scheme's launch action, so
      running from Xcode loads products locally — no App Store Connect record, no
      sandbox account, no paid-apps agreement, no money. One product,
      `com.budj.standard.yearly` at $199, because the identifier is the join key
      the server resolves a submitted transaction back to a plan with; a test on
      this side alone that drifted would fail only in production. **`plans.ts`
      still lists `starter` and `pro` and must be changed to match** — see 11.9.
      **It needed a membership exception.** `Budj/` is a synchronised root group,
      so the file joined the target automatically and was copied into the built
      `.app` — shipping the store configuration in the product. It is now
      excepted alongside `Info.plist`, and the built bundle was checked.

## 12. Bank connection

- [ ] 12.1 Request the authorisation URL from the server and present it with
      `@Environment(\.webAuthenticationSession)`, ephemeral session off.
- [ ] 12.2 Handle the callback agreed in 1.2 — and treat completion, cancellation,
      and dismissal identically: re-fetch status and render the server's answer.
- [ ] 12.3 `ConnectBankView` making no assumption that it is the first connection,
      so settings can reuse it.
- [ ] 12.4 Inline handling of `plan_limit_exceeded` without leaving the step.
- [x] 12.5 ~~Copy stating read-only access and revocability, and implying no
      ability to move money.~~ **Done**, and it is the real copy rather than a
      stand-in — three lines on `ConnectBankView`, each with a symbol as well as
      text: Budj reads accounts and transactions and asks for nothing else, it
      cannot move money through this connection, and access can be revoked from
      either end. The screen also says you sign in with your bank rather than
      with us, which is the thing people actually want to know before tapping.
      The symbols are `accessibilityHidden` — they repeat the sentence beside
      them, and announcing both reads every row twice.
- [ ] 12.6 Test: cancelling the session leaves the user on the bank step with no
      error that suggests they did something wrong.

## 13. Push

- [x] 13.1 ~~Push opt-in screen after the bank step, with a skip of equal
      prominence.~~ **Done, and not mocked** — push is client-only, so unlike
      billing and bank there was no server answer to stand in for. "Not now" is a
      `secondary` button rather than a text link, which is what "equal
      prominence" has to mean if it is to mean anything. The screen requests
      authorisation for real; both answers call back identically, because the
      router only records that the offer was made and iOS will not ask twice
      either way. Registering for remote notifications and forwarding the APNs
      token is 13.2–13.3 and is not here.
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
      **Partly done, and for a bigger reason than the voice rules.** The flow was
      written for a salaried consumer — the welcome screen's own example was
      "when your salary lands, move $200 to savings", which is the one case Budj
      is *not* for: a fixed amount on a fixed date is already solved by a bank
      automatic payment, for free. Budj exists because the amount and the timing
      are unknown in advance, which a scheduled transfer cannot express and a
      rule can. Welcome, paywall, bank and push now address somebody
      self-employed splitting irregular income for tax and GST. Still to do: the
      sign-in and biometric screens, and a pass at the largest type size, since
      the new copy is longer.
- [ ] 15.4 Run the app on an iPad simulator. `TARGETED_DEVICE_FAMILY` is `"1,2"`;
      onboarding stays single-column but must not look broken.
