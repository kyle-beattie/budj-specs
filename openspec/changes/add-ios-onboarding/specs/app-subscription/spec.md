## ADDED Requirements

### Requirement: Prices are rendered from StoreKit, never formatted by the app

The paywall SHALL display each product's `displayPrice` as supplied by StoreKit.
The app MUST NOT construct a price string from a numeric value or assume a
currency.

#### Scenario: The displayed price matches the App Store sheet

- **WHEN** the paywall renders a plan
- **THEN** the price shown is the product's `displayPrice` and matches what the
  purchase sheet subsequently shows

#### Scenario: No currency is hard-coded in the paywall

- **WHEN** the paywall's source is inspected
- **THEN** it contains no currency symbol, currency code, or numeric price
  formatting

### Requirement: A purchase is submitted to the server and the server's answer decides entitlement

After a successful StoreKit purchase the app SHALL submit the signed transaction
to the server and re-fetch onboarding status. It SHALL NOT treat StoreKit's local
entitlement as sufficient to proceed.

#### Scenario: Purchase then submit then refresh

- **WHEN** a purchase completes and verifies locally
- **THEN** the signed transaction is submitted to the server
- **AND** onboarding status is requested again before the step changes

#### Scenario: A locally-entitled user whom the server does not recognise stays on the paywall

- **WHEN** StoreKit reports a current entitlement but the refreshed status still
  reports `billing`
- **THEN** the paywall remains displayed

#### Scenario: The transaction is finished only after submission succeeds

- **WHEN** submission to the server fails
- **THEN** the transaction is left unfinished so that it is re-observed and
  re-submitted later

### Requirement: Transaction updates are observed from app start

The app SHALL observe StoreKit's transaction updates from launch rather than from
the paywall, so that a purchase completing outside the paywall is not lost.

#### Scenario: A purchase completing while backgrounded is submitted

- **WHEN** a transaction arrives while the paywall is not on screen
- **THEN** it is submitted to the server and status is refreshed

### Requirement: Restore is offered and recovers an existing subscription

The paywall SHALL provide a restore action that synchronises with the App Store
and resubmits the resulting transaction.

#### Scenario: Restoring on a new device grants access

- **WHEN** a user with an active subscription installs the app on another device
  and taps restore
- **THEN** the transaction is resubmitted and, once the server records
  entitlement, onboarding proceeds past billing

### Requirement: A subscription bound to another account is reported as such

The app SHALL explain that specific outcome rather than reporting a generic
purchase failure, where the server refuses a submission because the original
transaction identifier already belongs to a different account.

#### Scenario: Duplicate binding is explained

- **WHEN** the server refuses a submitted transaction as already bound elsewhere
- **THEN** the paywall states that the subscription is in use on another account,
  distinctly from a network or verification failure

### Requirement: The app never claims to cancel a subscription

The app SHALL NOT present a cancellation control and SHALL direct the user to App
Store subscription management instead, because no server capability to cancel
exists.

#### Scenario: Cancellation directs to the App Store

- **WHEN** a user looks for how to stop paying
- **THEN** they are directed to App Store subscription management

#### Scenario: No in-app cancellation is implied

- **WHEN** subscription-related copy is reviewed
- **THEN** nothing states or implies that Budj can cancel, pause, or refund the
  subscription
