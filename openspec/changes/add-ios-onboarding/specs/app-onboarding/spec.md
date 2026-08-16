## ADDED Requirements

### Requirement: The displayed step is the server's derived step

The app SHALL render the step reported by `GET /api/onboarding/status` and SHALL
NOT maintain its own record of progress through the server-visible steps. It MUST
NOT advance past a step on the strength of a local success alone.

#### Scenario: No local progress cursor exists

- **WHEN** the app's persisted state is inspected
- **THEN** nothing records which onboarding step the user has reached

#### Scenario: Status is refreshed after a step that changes server state

- **WHEN** a purchase is submitted successfully, or the bank authorisation
  session returns
- **THEN** the app requests status again and renders whatever it reports

#### Scenario: The app does not advance itself optimistically

- **WHEN** a purchase submission succeeds locally but the refreshed status still
  reports `billing`
- **THEN** the billing step remains displayed

### Requirement: Onboarding resumes where the server says, on every launch

A user SHALL be able to leave at any point and return to the same step, with no
step repeated and no work lost.

#### Scenario: Resuming after abandoning at the bank step

- **WHEN** a user subscribes, closes the app before connecting a bank, and
  relaunches
- **THEN** the bank step is shown
- **AND** no paywall is presented

#### Scenario: Force-quitting mid-step loses nothing but the step

- **WHEN** a user force-quits during the bank authorisation session and relaunches
- **THEN** the app shows the step the server reports, which may be `bank` or
  `ready` depending on whether the exchange completed

### Requirement: Client-only steps are shown once and record their own outcome

The app SHALL record locally that each client-only step was offered and SHALL NOT
offer it again on a later launch. Biometric unlock and push notification opt-in
are invisible to the server, so nothing else can remember for it.

#### Scenario: A declined biometric offer is not repeated

- **WHEN** a user declines biometric unlock and relaunches the app
- **THEN** the biometric opt-in step is not shown again

#### Scenario: A declined push offer is not repeated

- **WHEN** a user declines notifications and relaunches the app
- **THEN** the push opt-in step is not shown again, because the system will not
  re-prompt in any case

### Requirement: Push opt-in never blocks completion

The push step SHALL be skippable with an action of equal prominence to accepting,
and skipping it SHALL lead directly to the ready state.

#### Scenario: Skipping push completes onboarding

- **WHEN** a user with an active subscription and a connected bank skips the push
  step
- **THEN** the app enters the ready state
- **AND** the status response's outstanding-push indication does not hold them
  back

#### Scenario: The skip action is not visually subordinate

- **WHEN** the push step is displayed
- **THEN** the skip action is a button, not a de-emphasised text link presented as
  a concession

### Requirement: The step order places billing before bank

The app SHALL present the paywall before the bank connection, matching the
server's derived order, and SHALL NOT offer a bank connection to a user without
an active entitlement.

#### Scenario: An unsubscribed user is not shown the bank step

- **WHEN** status reports `billing`
- **THEN** the paywall is displayed and no bank connection can be started from it

### Requirement: Entitlement lost during onboarding returns the user to billing

The app SHALL return to the billing step and say why, if any request fails with
the subscription-required error while onboarding is in progress.

#### Scenario: A refund mid-flow returns to the paywall

- **WHEN** the user is on the bank step and a request is refused with
  `subscription_required`
- **THEN** the billing step is displayed with an explanation that the subscription
  is no longer active
