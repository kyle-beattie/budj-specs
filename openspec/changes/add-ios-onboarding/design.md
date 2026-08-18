## Context

The iOS app is six files: an entry point, a placeholder `ContentView`, a launch
screen held for a fixed 1.2 seconds, and the brand mark drawn as vectors. The
server it talks to does not exist yet either, but its contract does — the
`add-onboarding` change specifies six capabilities in detail, and D14 of that
design commits to publishing an OpenAPI document and conformance vectors as a
tagged artifact precisely so this repository can be built against it without
seeing the server's source.

So this change has an unusual amount of certainty for a first change. The
endpoints, the error codes, the ordering, and the scope set are all decided. What
is undecided is everything about the app: how it is laid out, how state moves
through it, what a component is, and what happens on a cold launch when the
network is gone.

The author of this app is a React developer. That is a design input, not a
biography: SwiftUI's model is close enough to React that the instincts transfer,
and the places where the instincts are *wrong* are the places bugs will cluster.
This document maps the two explicitly rather than leaving it to be rediscovered.

One more constraint shapes the layout. `Budj/` is declared as a
`PBXFileSystemSynchronizedRootGroup` in `project.pbxproj` — Xcode 16's file
system synchronised groups — which means the folder tree on disk *is* the project
structure and new files join the target with no project-file edit. Reorganising
now costs nothing. (`CLAUDE.md` currently says the opposite; correcting it is a
task here.)

```
   THE WHOLE OF THIS CHANGE

   cold launch ──► session? ──no──► sign in ──► paywall ──► Face ID? ──►
                      │                                                 │
                     yes                                                ▼
                      │                                          connect bank
                      ▼                                                 │
              GET /onboarding/status ◄───── re-fetched after ───────────┤
                      │                     every server-state change   │
                      │                                                 ▼
                      └──► billing | bank | ready ◄──────────────  push opt-in
                                              │                    (skippable)
                                              ▼
                                        the app (out of scope)
```

## Goals / Non-Goals

**Goals**

- A person can get from first launch to `ready`, and can abandon at any point and
  resume from the same place on the next launch.
- The app never holds an opinion about which onboarding step someone is on. The
  server derives it; the app renders it.
- A component layer exists that the remaining four feature areas can be built on
  without revisiting this decision.
- Every component is reviewable in isolation, in both appearances, at accessible
  type sizes, without running the flow.
- The app is buildable and testable against the published contract before the
  server exists.

**Non-Goals**

- The four main tabs, the rule builder, approval review, and account deletion.
  This change ends at `ready`.
- Offline capability beyond degrading honestly. There is nothing useful to do
  offline during onboarding — every step needs a network.
- Any local database. Nothing in onboarding is worth persisting except the
  session, which belongs in the Keychain.
- Per-payment biometric approval. Dropped from the product in `add-onboarding`
  D10, and no client affordance should imply it exists.
- iPad-specific layout. `TARGETED_DEVICE_FAMILY` is `"1,2"` and the app must not
  break on iPad, but onboarding is a single-column flow and stays one.

## Decisions

### D1. The structure is by feature, one type per file, and components rather than computed properties

```
Budj/
├── App/
│   ├── BudjApp.swift              @main, environment wiring, nothing else
│   ├── RootView.swift             the launch gate and top-level router
│   └── AppPhase.swift             enum: launching | onboarding | ready | mustUpdate
├── DesignSystem/
│   ├── Tokens/
│   │   ├── BudjColor.swift        asset-catalogue colours, named semantically
│   │   ├── BudjSpacing.swift      the 4pt scale
│   │   ├── BudjRadius.swift       sm 12 / md 20 / lg 28 / xl 32 / full
│   │   ├── BudjTypography.swift   Font roles mapped onto Dynamic Type
│   │   └── BudjMotion.swift       standard/spring curves, three durations
│   ├── Components/                one file per component, no exceptions
│   │   ├── BudjButton.swift       BudjButtonStyle.swift
│   │   ├── BudjCard.swift         BudjBadge.swift
│   │   ├── BudjTextField.swift    BudjToggleRow.swift
│   │   ├── BudjListRow.swift      BudjBankRow.swift
│   │   ├── GlassSurface.swift     MoneyText.swift
│   │   └── StepScaffold.swift     the shared onboarding screen shell
│   ├── Modifiers/
│   │   ├── PressScale.swift       CardSurface.swift, GlassBackground.swift
│   └── Gallery/
│       └── ComponentGallery.swift every component, every state, one screen
├── Features/
│   ├── Launch/                    LaunchScreenView, LaunchGateModel
│   ├── Onboarding/
│   │   ├── OnboardingFlowView.swift    the router
│   │   ├── OnboardingModel.swift       the only stateful object in the flow
│   │   ├── OnboardingStep.swift        the step enum
│   │   └── Screens/                    one file per screen
│   │       ├── WelcomeView.swift       SignInView.swift
│   │       ├── EmailSignInView.swift   PaywallView.swift
│   │       ├── BiometricOptInView.swift ConnectBankView.swift
│   │       └── PushOptInView.swift     MustUpdateView.swift
│   └── Placeholder/               where `ready` lands until the tabs exist
├── Core/
│   ├── Networking/                BudjAPI, Endpoint, APIError, ClientBuild
│   ├── Session/                   SessionStore, KeychainStore, BiometricGate
│   ├── Purchases/                 StoreKit wrapper
│   └── Models/                    OnboardingStatus, Plan, BankConnection, Money
└── Branding/                      BudjMark, BudjMarkShape (existing)
```

Two rules do most of the work. **One type per file**, so a file name is a
component name and the tree is a component index. **Extract a `View` struct
instead of a `private var someView: some View`** — a computed property that
returns `some View` is the SwiftUI equivalent of defining a component inside
another component's render function, and it has the same costs: nothing can
preview it, nothing can test it, and nothing can reuse it. A React developer's
reflex to pull out a component is exactly right here; the only difference is that
the new component goes in a new file.

The permitted exception is a small `private var` used purely for structural
readability that would be acceptable inlined into `body` — a toolbar's content,
say. If it is a thing with a name and states, it is a component.

### D2. The React translation, stated once so it is not re-derived per file

This is the table to consult when the SwiftUI equivalent of a familiar move is
not obvious.

| React                                | SwiftUI                                                          |
|--------------------------------------|------------------------------------------------------------------|
| `function Card({ title })`           | `struct BudjCard: View { let title: String }`                     |
| props                                | `let` properties on the view struct                               |
| `children`                           | `@ViewBuilder let content: () -> Content`                         |
| named slots / render props           | several `@ViewBuilder` properties (`header:`, `footer:`)          |
| `useState`                           | `@State private var`                                              |
| lifting state up                     | `@Binding var`, or a closure property `let onSubmit: () -> Void`  |
| callback props (`onChange`)          | closure properties, same idea, same name                          |
| `useContext` / `<Provider>`          | `@Environment(...)` and `.environment(...)`                       |
| defining a context value             | `@Entry var budjAPI: BudjAPI` in an `EnvironmentValues` extension |
| custom hook (`useOnboarding()`)      | an `@Observable` class injected through the environment           |
| `useReducer`                         | `@Observable` model holding an enum state plus intent methods     |
| `useEffect(fn, [])`                  | `.task { }` — and it cancels itself on disappear                  |
| `useEffect(fn, [dep])`               | `.task(id: dep) { }` or `.onChange(of: dep) { }`                  |
| `useMemo`                            | usually nothing; occasionally `@State` used as a cache            |
| `key` prop to force a remount        | `.id(value)` — the exact analogue, including the state reset      |
| `React.memo`                         | `Equatable` conformance on the view                               |
| CSS modules / styled-components      | `ViewModifier` plus a `View` extension, e.g. `.cardSurface()`     |
| a variant prop (`<Button kind="…">`) | an enum property, or a `ButtonStyle` chosen by the caller         |
| design tokens as CSS custom props    | `BudjColor` / `BudjSpacing` / `BudjRadius` static members         |
| Storybook                            | `#Preview` per component, plus `ComponentGallery`                 |
| React Router                         | `NavigationStack(path:)` with a typed route enum                  |
| controlled input                     | `TextField("…", text: $binding)`                                  |
| barrel file (`components/index.ts`)  | nothing — the module is the namespace                             |

The differences that actually bite:

- **`body` runs constantly and that is fine.** Views are value types, cheap to
  create, and re-created on every state change. There is no `useCallback`
  problem, because a closure property is not a dependency that invalidates a
  memo. Do not optimise this preemptively.
- **`@State` does not reset when the parent re-renders.** In React, a component
  that changes position in the tree remounts and loses state. In SwiftUI, state
  survives as long as structural identity does. `.id(…)` is the deliberate reset,
  and it is the tool for "clear this form after a successful submit".
- **`if`/`else` branches are separate identities**, so state does not carry
  across them — the same trap as conditionally rendering two different components
  in React, and the same fix.
- **The environment is not props, and it is not typed at the call site.** A view
  reading `@Environment(OnboardingModel.self)` will crash at runtime if nobody
  provided one — the same failure as a missing Provider, without a helpful error.
  Provide everything in `BudjApp` and nowhere else, so there is one place to look.
- **There is no cleanup function.** `.task` is cancelled automatically when the
  view goes away; structured concurrency does what `useEffect`'s return value
  does, without being asked.

### D3. Three layers, and a hard rule about which knows about the API

```
   Tokens          colour, spacing, radius, type, motion
     │             knows nothing; is data
     ▼
   Components      BudjButton, BudjCard, BudjTextField, StepScaffold…
     │             knows tokens; takes props and closures
     │             MUST NOT import the API, the session, or a model
     ▼
   Feature views   SignInView, PaywallView, ConnectBankView…
     │             composes components; reads models from the environment
     ▼
   Models          OnboardingModel and friends: @Observable, testable,
                   no SwiftUI types in their public surface
```

The one rule that keeps this honest: **a component in `DesignSystem/` may not
know that a server exists.** It takes values and closures. The moment
`BudjBankRow` fetches something, it stops being previewable, stops being
reusable, and the gallery stops being able to render it.

This is the same discipline as presentational-versus-container components, and it
is worth keeping for the same reason: the layer that is hardest to test is the
layer that talks to the network, and it should be as thin and as far from the
pixels as possible.

### D4. The asset catalogue is the source of truth for the visual language

Colour comes from the asset catalogue and nowhere else. It carries a warm
off-white `#EDF8E5` and a warm near-black `#1A140D` background pair, and the mark
in cyan `#00C5FA`, amber `#FFB700` and green `#00ED9A`. Every colour the app uses
is a named set in the catalogue with both appearances defined, surfaced through
`BudjColor` — so a colour has exactly one definition and light mode is impossible
to forget.

**Light appearance is supported.** The catalogue already defines it, and shipping
a dark-only app that has light colours sitting unused is a decision nobody made.

The rest of the visual language is decided here, because there is no external
document that holds it:

- **Corners are a full-radius system.** Buttons and pills fully rounded; cards and
  sheets 28–32pt. Never sharp, never barely-rounded.
- **Spacing is a 4pt scale**, named rather than numeric, and nothing uses a value
  off it.
- **Glass is the signature surface**, mapped to iOS 26's native glass APIs rather
  than hand-built blur. Reserved for surfaces floating over content — sheets,
  toasts, the tab bar, the primary action. Dense lists use solid surfaces, where
  blur costs legibility.
- **Motion** has one standard curve and one springier curve for toggles and sheet
  presentation, with three durations: 120ms for press feedback, 200ms default,
  340ms for sheets and transitions.
- **Type is the system font**, with `.monospacedDigit()` wherever figures need to
  line up. No custom faces are bundled; if that changes it is a deliberate
  decision with a licensing answer attached, not a drive-by.
- **Icons are real SF Symbols.**
- **Copy is sentence case, second person, no emoji, no exclamation points.**
  Uppercase tracked text only in small badge and eyebrow labels.

`ComponentGallery` (D14) is what keeps this honest. A written palette drifts; a
screen rendering every component in both appearances does not, because the drift
is visible the moment someone opens it.

### D5. The onboarding step is the server's answer, and the app is a router

`GET /api/onboarding/status` returns the first unsatisfied step. The app renders
that step. It does not keep a local cursor, it does not advance itself after a
successful purchase, and it does not remember across launches where it was.

```
   OnboardingModel
     step: .billing | .bank | .ready        ← from the server, every time
     pushOutstanding: Bool                  ← advisory, from the same response

   after any step that changes server state:
     purchase submitted ──► refresh()
     bank returned      ──► refresh()
     (Face ID, push)    ──► no refresh; the server has no opinion on these
```

The reason is D1 of the server design: the step is derived from stored facts, so
it is correct by construction and cannot drift. Any local mirror of it can, and a
local mirror that says `bank` when the server says `billing` produces a person
stuck on a screen that will never let them through.

The cost is a round trip after each step, which is the correct trade for a flow
someone traverses once.

**Client-only steps interleave.** Face ID and push are invisible to the server, so
the app owns their placement. Face ID is offered immediately after sign-in, while
the session is the thing on the user's mind. Push is offered after the bank
connects, where the reason for it is concrete. Neither step is ever shown twice:
both record their outcome locally, since a declined permission asked again on
every launch is hostile and, for notifications, the system will not re-prompt
anyway.

**Face ID comes before the paywall, not after it.** An earlier draft of this
section put the paywall first on the grounds that the server requires billing
before bank unconditionally — but that is a statement about the server's step
order, and Face ID is not one of the server's steps. Nothing prevents the app
from asking about the session it has just established before it asks for money,
and meeting the price as the very first thing that happens after signing in is
worse UX for no gain. The order is:

```
   welcome → signIn → biometrics → billing → bank → push → ready
             └ app ┘  └── app ──┘  └──── server ───┘ └app┘
```

The Face ID step is keyed off **the first server step after signing in**, whatever
it is, rather than off `billing` specifically. Keying it to `billing` would skip
the offer entirely for somebody who reinstalls, signs in, and lands directly on
`bank` because they already subscribed.

### D6. The launch gate replaces the timed hold

`RootView` currently sleeps for 1.2 seconds and fades. That is a placeholder that
happens to look like a product decision. It becomes a real gate over real work:

```
   LaunchScreenView is shown for as long as this takes, and no longer:

   1. read the session from the Keychain
   2. if it is biometry-protected, ask for Face ID
   3. if there is a session, GET /api/onboarding/status
   4. route: mustUpdate | onboarding(step) | ready | signIn

   with a floor of ~0.6s so a fast path does not flash,
   and a ceiling after which step 3's failure is surfaced rather than waited on
```

Three failure modes need answers, and the answers are what makes this a design
decision rather than an implementation detail:

- **No session.** Go to welcome. This is the ordinary first launch.
- **Session present, status request fails (offline, server down).** Show a
  retryable "can't reach Budj" state rather than assuming a step. Assuming
  `billing` would show a paywall to a paying customer, which is the worst
  available guess.
- **Session present but rejected (401).** Clear the Keychain item and go to
  welcome. A refresh token that Supabase no longer honours is not recoverable in
  the app.
- **Any response carrying the unsupported-client code.** Route to `MustUpdateView`
  from wherever you are, including here. This outranks every other state,
  because by definition nothing else the app does is trustworthy.

### D7. The Apple identity token and the Apple authorization code are different things, and both are load-bearing

`SignInWithAppleButton` yields an `ASAuthorizationAppleIDCredential` carrying
both. They go to different places and neither substitutes for the other:

```
   identityToken ──────► Supabase signInWithIdToken     (never touches our server)
   authorizationCode ──► POST our server                (never touches Supabase)
                              │
                              └─► exchanged with Apple for a refresh token,
                                  encrypted, stored, and needed exactly once:
                                  at account deletion, months from now
```

The code is single-use and expires in minutes, so there is no later opportunity
to collect it — this is the whole reason the server change captures it during
onboarding (server D13). Getting it wrong yields an app that works perfectly and
an account that can never be properly deleted, discovered in
`add-account-deletion` when it is far too late.

Per the server's `identity` spec, a failed exchange must not fail sign-in: post
it, ignore a failure beyond logging it, and let the user through.

The credential also carries the user's name **only on the very first
authorisation** and never again. It is forwarded with the sign-in so the profile
can be seeded, and the app must not attempt to reconstruct a display name from an
email address — particularly not from a `@privaterelay.appleid.com` address,
which the server spec explicitly forbids.

### D8. The session lives in the Keychain behind biometry, and biometric failure falls back to signing in

Face ID here unlocks a locally-held refresh token. It is not a payment
authorisation and the server cannot verify it (server D10). The app should not
imply otherwise — no "confirm with Face ID" language anywhere near money.

The Keychain item is written with `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`
and, when the user opts in, an access control requiring biometry. If they decline,
the same item is written without the biometry flag; the session still persists and
relaunching still resumes. Declining is a supported configuration, not a degraded
one.

`.biometryCurrentSet` over `.biometryAny`: enrolling a new face or fingerprint
invalidates the item, and the user signs in again. That is the safer default for a
finance app and the failure is legible — you added a face, you signed in again —
where the alternative silently extends trust to a biometric the user never
enrolled for this purpose.

**Failure falls back, it does not lock.** A cancelled or failed Face ID prompt
returns to sign-in. There is no retry counter and no lockout screen, because there
is no local data to protect beyond a token that can be reissued by signing in.

### D9. StoreKit shows Apple's price string, and the server is told rather than asked

Two rules, both narrow and both easy to violate:

- **`Product.displayPrice` is the only price the app ever renders.** It is already
  localised and already carries the correct currency for the user's storefront. A
  price formatted by the app is a price that will be wrong for somebody, and a
  mismatch between the paywall and the App Store sheet is a review rejection.
- **Entitlement comes from the server, not from `Transaction.currentEntitlements`.**
  StoreKit's local view is a convenience; the server's cached entitlement, fed by
  App Store Server Notifications, is the authority the API gates on. The app
  purchases, submits the signed transaction, and then calls `refresh()`. If the
  server disagrees with StoreKit, the server is right.

Restore is a required affordance (`AppStore.sync()`, then resubmit). It is also
the recovery path for a purchase whose submission failed — the transaction stays
unfinished and is re-observed on next launch. `Transaction.updates` must be
observed from app start, not from the paywall's `.task`, or a purchase that
completes while the app is backgrounded is lost.

Two server behaviours the paywall must render as distinct outcomes rather than a
generic failure: a transaction already bound to a different account, and
`plan_limit_exceeded` later in the flow.

### D10. Bank connection is a hand-off, and the server's answer is the outcome

The app requests an authorisation URL, presents it, and comes back. It never sees
an Akahu token and never calls Akahu — that is a hard requirement of the server's
`bank-connections` spec, not a preference.

Presentation uses `@Environment(\.webAuthenticationSession)`, the SwiftUI-native
`ASWebAuthenticationSession`, which keeps this free of UIKit and gives the system
consent sheet for free. `prefersEphemeralWebBrowserSession` is left **off**: the
user is signing into their bank, and a shared session with Safari is the
behaviour they expect from a bank hand-off.

**The return is not the outcome.** The web session can complete, be cancelled, be
dismissed by a swipe, or return while the server's code exchange is still in
flight. None of those tell the app whether a connection now exists. So on any
return — including cancellation — the app re-fetches onboarding status and
connections, and renders what the server says. This makes the flow correct
regardless of how the redirect is delivered, which matters because how it is
delivered is X2.

Connecting a second bank re-enters the identical flow. The screen is written once
and reused from settings later, so it must not assume it is the first connection.

### D11. Push is asked for once, after the bank, and skipping is a first-class outcome

`ready` does not require push (server D12), so the app must not either. The
opt-in appears after the bank connects, with a plain statement of what it is for —
a rule matched and needs your approval — and a skip that is a real button, not a
smaller-text apology.

If the user allows it, the APNs token is registered. If they decline, nothing is
registered and the app proceeds. The system prompt appears once per install and
cannot be re-shown, so the app records that it asked and never asks again; the
recovery path is a settings row that deep-links to iOS Settings, which belongs to
a later change.

Registration needs an application delegate to receive the token
(`UIApplicationDelegateAdaptor`). That is the single unavoidable UIKit touchpoint
in this change and it should stay a handful of lines whose only job is to forward
the token.

### D12. The API client is one type, and its errors are an enum the UI switches over

Every request carries the build identifier, because the server treats a request
without one as unsupported rather than exempt. That makes it a property of the
client, set once, never per-call — there is no code path that can forget it.

```
   enum APIError {
     case unsupportedClient        ──► MustUpdateView, from any screen
     case subscriptionRequired     ──► back to the paywall, mid-session
     case planLimitExceeded        ──► inline on the connect screen
     case unauthorized             ──► clear session, back to welcome
     case network(…)  case server(…)  case decoding(…)
   }
```

The first two are cross-cutting: they can arrive in response to any call, and the
answer is the same wherever they arrive. They are handled once, at the top, rather
than by every call site — the analogue of an axios interceptor, and for the same
reason.

`subscriptionRequired` mid-session is a real case, not a theoretical one: a refund
or an expiry notification can revoke entitlement while the app is open, and the
server also revokes the Akahu token when that happens. The app returns to the
paywall and says so plainly.

**Conformance vectors are a test fixture, not documentation.** The published
artifact's money vectors are checked into the app's test target and asserted
against the app's own decoder. That is the only mechanism that catches client
drift, and it costs one test.

### D13. Onboarding is a switch, not a navigation stack

There is no back button in onboarding. You cannot return to the paywall after
buying, and you cannot un-connect a bank by swiping back. Modelling it as a
`NavigationStack` would create a back stack that has to be suppressed everywhere,
which is a fight with the framework.

Instead `OnboardingFlowView` switches over the step and applies a transition, with
`.animation(value: step)` so the animation is driven by the value rather than
implicitly. Where a step needs internal depth — email sign-in behind the provider
buttons — that step owns a local `NavigationStack` or presents a sheet, and it
does not leak.

`NavigationStack(path:)` with a typed route enum and one
`navigationDestination(for:)` per type is the pattern for the tabs, later. It is
recorded here so the two are not mixed: never `NavigationLink(destination:)`
alongside `navigationDestination(for:)` in one hierarchy.

Every onboarding screen is built on one `StepScaffold` component taking a title,
a body, and a primary and optional secondary action. Six screens sharing one
shell is what makes the flow feel like one flow, and it is where the visual rules
— safe areas, the glass primary action, spacing rhythm — are enforced once.

### D14. The component gallery is the Storybook, and it is shipped in the app target

`#Preview` covers a component in isolation, but previews are invisible unless
someone opens the file. `ComponentGallery` is a scrollable screen rendering every
component in every meaningful state, reachable in debug builds only.

It earns its keep three ways: reviewing the design language without navigating a
flow to find a component; checking light and dark side by side; and — the reason
it is worth building — dragging the type-size slider to the accessibility sizes
and seeing which components break, which is otherwise something nobody does.

### D15. The accessibility floor is part of the component contract

Not a later pass. A component that fails these is not finished:

- **Dynamic Type throughout.** Semantic fonts (`.body`, `.headline`) and no
  hard-coded sizes. Where a display size is genuinely needed, `.font(.body
  .scaled(by:))` — never a fixed point size.
- **Every button has a text label**, even where the visual is icon-only; use
  `.labelStyle(.iconOnly)` to keep the visual while keeping the label for
  VoiceOver. Bank tiles are buttons with the bank's name, not tappable images.
- **Reduce Motion** replaces the step transitions and the launch fade with
  opacity. The launch mark's animation is a decorative flourish and must not
  play at all when Reduce Motion is on.
- **Never `onTapGesture` for something tappable.** Use a `Button`.
- **Connection state is not conveyed by colour alone** — a connecting bank shows a
  progress view and a connected one shows a checkmark, which also satisfies
  `accessibilityDifferentiateWithoutColor`.

### D16. There is no Supabase SDK; one endpoint is called directly and the rest goes through our server

Resolves X1. `supabase-swift` is **not** adopted. The dependency count stays at
zero.

What made this a decision rather than a coin flip is the published contract. The
server already owns the parts a client library would have earned its place on:

| Need                        | Where it goes                                    |
|-----------------------------|--------------------------------------------------|
| Email/password sign-in      | `POST /api/auth/sign-in`   — our server           |
| Registration                | `POST /api/auth/sign-up`   — our server           |
| Refreshing an expired token | `POST /api/auth/refresh`   — our server           |
| Apple/Google identity token | `POST {SUPABASE_URL}/auth/v1/token?grant_type=id_token` |

Only the last row talks to Supabase, and it is one `URLSession` call carrying the
provider, the identity token, and `data.full_name` when Apple supplies it (D7).
Session persistence is the Keychain work in 7.1/7.2 either way — a library would
have arrived with its own session store to keep in step with ours, which is a cost
rather than a saving.

The app therefore needs `SUPABASE_URL` and the Supabase **anon** key in its
configuration. The anon key is a publishable identifier, not a credential, so this
does not violate 15.2 — but it belongs in a configuration file rather than
inlined in a call site, and no service-role key ever ships.

The direct call is the one place `BudjAPI`'s rules do not apply: it carries no
`Authorization` header and no `X-Client-Build`, because it is not our server. It
lives in `Core/Session/` rather than `Core/Networking/` so that separation is
visible in the tree.

Revisit if Google is chosen natively (X3) and the OAuth dance turns out to want
more of GoTrue than one endpoint.

## Risks / Trade-offs

**The two silent failures.** Sending Apple's identity token where the
authorization code belongs, and omitting the build identifier, both produce an app
that appears to work. Neither is detectable by using the product. Both are covered
by a single assertion in the test suite, which is why the tasks put them there
rather than in a review checklist.

**A round trip per step.** Deriving the step server-side means the flow is
network-bound at every boundary. On a poor connection the app will feel slow at
exactly the moment a new user is deciding whether to continue. Accepted: the
alternative is a local cursor that can disagree with the server, and a person
stuck behind a step they have already completed is worse than a person waiting
two seconds. The mitigation is that the button shows progress in place rather than
blocking the screen.

**Face ID is theatre against a determined attacker**, and the design says so. It
protects a token on a locked device, nothing more. The risk is that the *product*
comes to imply otherwise — a "secured by Face ID" line on the paywall would be a
misrepresentation. Server D10 records what actually bounds the damage: Akahu's
bank-enforced consent limits, which do not exist yet.

**There is no design document outside the code.** The visual language is the
asset catalogue plus D4 plus whatever the components actually do, and those three
can disagree. This is a deliberate trade — an external spec that drifts out of
date is worse than none, and the last one did exactly that — but it means the
gallery is load-bearing rather than a nicety. If 5.1 is skipped, nothing enforces
the language at all.

**A component layer built for six screens may not fit thirty.** Real risk, and
the reason the layering rule in D3 is narrow rather than elaborate: tokens,
primitives, and a rule about what a primitive may not know. Everything beyond that
is deferred until a second feature area asks for it.

## Migration Plan

There is nothing to migrate — no users, no persisted state, no released build.
The only existing code affected is the six files already present, and only two of
them change meaningfully:

1. Move the existing files into the new tree. No project-file edits required;
   the synchronised root group picks them up.
2. `RootView` loses its fixed `Task.sleep` and gains the launch gate (D6).
   `LaunchScreenView` and the brand mark are unchanged.
3. `ContentView` is deleted. It is a placeholder and its name means nothing; what
   replaces it is `Features/Placeholder/`.
4. Correct the `CLAUDE.md` claim that new sources must be registered in
   `project.pbxproj`, and record the design-source split from D4.

## Open Questions

1. ~~**The Supabase Swift client is a third-party dependency.**~~ **Answered — see
   D16.** Hand-rolled. Refresh, password sign-in and registration all have server
   routes already, so the only call left to Supabase is a single id-token
   exchange, and the session store was ours to write regardless.
2. **How does the app learn the Akahu redirect completed?** The server owns the
   redirect URI and performs the code exchange, but neither the server's
   `bank-connections` spec nor its design says what the server redirects *to*
   afterwards. Options are a custom scheme, a universal link, or a terminal page
   the user dismisses. D10 makes the app correct under all three by treating the
   server's status as the outcome, but the callback URL has to be agreed with the
   server change before either side can be built. **This is a gap in
   `add-onboarding`, not only here.**
3. ~~**Native Google sign-in normally means the GoogleSignIn SDK**~~
   **Deferred — Google is not in this change.** The two routes are the SDK,
   which is a third-party dependency and would cost the position D16 takes, and
   Supabase's OAuth flow through `ASWebAuthenticationSession`, which needs no
   SDK and reuses the machinery D10 builds for Akahu, at the cost of a browser
   round trip. Neither is chosen: Apple plus email satisfies review, and
   spending the zero-dependency position under onboarding's deadline is how it
   gets spent by accident. Revisit after onboarding ships, when the OAuth route
   is the cheaper of the two because task 12 will already have built the web
   authentication session it needs.
4. **What does the app show for a broken bank connection?** Open question 5 of
   the server change, unresolved there, and the app is where it becomes visible.
   Out of scope for onboarding — a connection cannot break during the minute it
   takes to create it — but `ConnectBankView` is reused from settings later, and
   whoever specifies the recovery flow should start from that screen.
5. ~~**Is there a welcome screen before sign-in, or does the app open on
   sign-in?**~~ **Answered — yes.** `WelcomeView` is the first screen: what Budj
   does, then two actions of equal weight, "Create an account" and "I already
   have an account". Both open the same screen and differ only in which mode it
   starts in. It is the entry point rather than a step completed once, so it is
   not recorded and not skipped — signing out returns you to it, and so does
   relaunching while signed out.
