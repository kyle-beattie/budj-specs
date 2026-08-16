## ADDED Requirements

### Requirement: The launch screen is held by work, not by a timer

The app SHALL display the launch screen until session restoration and the first
onboarding status request have resolved, and SHALL NOT hold it for a fixed
interval. A short minimum duration MAY be applied so that a fast path does not
flash.

#### Scenario: A fast launch is not padded to a fixed delay

- **WHEN** a stored session is restored and status returns in under the minimum
  duration
- **THEN** the launch screen is dismissed at the minimum duration, not at a
  hard-coded interval unrelated to the work

#### Scenario: A slow launch is not cut short

- **WHEN** the status request takes longer than the minimum duration
- **THEN** the launch screen remains until the request resolves or fails, and the
  app never renders a step it has not been told to render

### Requirement: The routing decision on launch has a defined answer for every outcome

The app SHALL resolve exactly one destination from launch: an update prompt, a
sign-in screen, an onboarding step, the ready state, or a retryable connection
failure. It MUST NOT guess a step when status is unavailable.

#### Scenario: No stored session goes to sign-in

- **WHEN** the app launches with no session in the Keychain
- **THEN** the sign-in entry point is shown

#### Scenario: A rejected session is cleared rather than retried

- **WHEN** a restored session is refused by the server as unauthorized
- **THEN** the stored session is removed from the Keychain and the sign-in entry
  point is shown

#### Scenario: An unreachable server does not produce a guessed step

- **WHEN** a session is restored but the status request fails for network reasons
- **THEN** a retryable connection failure is shown
- **AND** no onboarding step, paywall, or ready state is displayed

#### Scenario: An unsupported build outranks every other destination

- **WHEN** any response during launch carries the unsupported-client error code
- **THEN** the update prompt is shown regardless of session or onboarding state

### Requirement: A biometric-protected session is unlocked before use

Where the user has enrolled biometric unlock, the app SHALL require it to read
the stored session, and a failed or cancelled prompt SHALL return the user to
sign-in rather than to a locked state.

#### Scenario: Successful unlock resumes the session

- **WHEN** the user passes the biometric prompt at launch
- **THEN** the stored session is read and the app routes on onboarding status

#### Scenario: Cancelling the prompt falls back to sign-in

- **WHEN** the user cancels or fails the biometric prompt
- **THEN** the sign-in entry point is shown
- **AND** no lockout, retry counter, or passcode screen is presented

#### Scenario: Biometric enrolment changing invalidates the stored session

- **WHEN** the device's biometric enrolment has changed since the session was
  stored
- **THEN** the stored item is no longer readable and the user signs in again

### Requirement: The launch animation respects Reduce Motion

The launch mark's animation SHALL NOT play when the system Reduce Motion setting
is enabled, and the handover to the app SHALL use opacity in that case.

#### Scenario: Reduce Motion suppresses the mark animation

- **WHEN** the app launches with Reduce Motion enabled
- **THEN** the mark is drawn in its final state without animating
- **AND** the transition away from the launch screen is a cross-fade
