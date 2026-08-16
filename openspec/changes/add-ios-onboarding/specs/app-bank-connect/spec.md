## ADDED Requirements

### Requirement: The app presents the server's authorisation URL and never contacts Akahu itself

The app SHALL request an authorisation URL from the server and present it in a
system web authentication session. It MUST NOT construct an Akahu URL, hold an
Akahu token, or call any Akahu endpoint.

#### Scenario: The URL comes from the server

- **WHEN** a subscribed user starts a bank connection
- **THEN** the app requests the authorisation URL from the server and presents
  exactly what it was given

#### Scenario: No Akahu credentials exist in the app

- **WHEN** the app's source and its stored data are inspected
- **THEN** they contain no Akahu token, application secret, or scope list

### Requirement: The outcome of a connection is what the server reports, not what the web session returned

The app SHALL re-fetch onboarding status and the connection list on any return
from the authorisation session — success, cancellation, or dismissal — and SHALL
render that answer rather than what the session appeared to do.

#### Scenario: A completed session is confirmed with the server

- **WHEN** the authorisation session returns a callback
- **THEN** the app re-fetches status rather than assuming the connection exists

#### Scenario: A cancelled session is also confirmed

- **WHEN** the user dismisses the authorisation session without completing it
- **THEN** the app re-fetches status and remains on the bank step if no connection
  was created

#### Scenario: A callback arriving before the exchange completes is tolerated

- **WHEN** status still reports `bank` immediately after a callback
- **THEN** the bank step remains and the user can retry, with no error implying
  the attempt was invalid

### Requirement: The bank step explains what access is being granted

Before hand-off, the screen SHALL state that access is read-only and that the
connection can be revoked, consistent with the scopes actually requested.

#### Scenario: Read-only access is stated

- **WHEN** the bank step is displayed
- **THEN** it states that Budj reads accounts and transactions and that the
  connection can be revoked

#### Scenario: No payment capability is implied

- **WHEN** the bank step's copy is reviewed
- **THEN** nothing implies Budj can move money yet, since the payments scope is
  not requested by this flow

### Requirement: The same screen connects the first bank and any additional bank

The connection screen SHALL make no assumption that it is being used during
onboarding, so that it can be reused to add a further institution later.

#### Scenario: A second connection uses the same flow

- **WHEN** a user with one connection starts another
- **THEN** the same screen and the same authorisation hand-off are used

#### Scenario: A connection limit is reported inline

- **WHEN** the server refuses with `plan_limit_exceeded`
- **THEN** the screen reports that the plan's connection limit is reached, without
  leaving the step

### Requirement: Connection progress is conveyed without relying on colour

Each institution's state SHALL be distinguishable by shape or symbol as well as
colour, and each SHALL be an accessible button carrying the institution's name.

#### Scenario: States are distinguishable without colour

- **WHEN** an institution is connecting or connected
- **THEN** a progress indicator or a checkmark distinguishes the state
  independently of any colour change

#### Scenario: Institutions are buttons with names

- **WHEN** VoiceOver traverses the bank list
- **THEN** each item is announced as a button with the institution's name
