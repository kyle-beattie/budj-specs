## ADDED Requirements

### Requirement: Every request carries the client build identifier

The build identifier SHALL be applied by the networking layer to every outbound
request, with no per-call opportunity to omit it, because the server treats a
request without one as an unsupported client rather than an exempt one.

#### Scenario: The identifier cannot be forgotten at a call site

- **WHEN** any request is constructed through the API client
- **THEN** it carries the build identifier without the caller supplying it

#### Scenario: The identifier reflects the actual build

- **WHEN** the identifier is read
- **THEN** it derives from the app's bundle build number rather than a constant
  written by hand

### Requirement: Server errors are decoded into an enumeration the interface can act on

The client SHALL map the server's error codes onto distinct cases, so that
callers switch over an outcome rather than parse a message.

#### Scenario: Distinct causes produce distinct cases

- **WHEN** the server responds with the subscription-required, plan-limit, or
  unsupported-client error code
- **THEN** each decodes to its own case, distinguishable from a network failure
  and from each other

#### Scenario: An unrecognised error code degrades rather than crashes

- **WHEN** the server returns an error code the app does not know
- **THEN** it is surfaced as a general server failure and the app continues to
  function

### Requirement: Unsupported client and lost entitlement are handled once, centrally

These two outcomes SHALL be handled in one place rather than at every call site,
because they can arrive in response to any request and have the same answer
wherever they arrive.

#### Scenario: An unsupported build routes to the update prompt from anywhere

- **WHEN** any request fails with the unsupported-client code
- **THEN** the app presents the update prompt regardless of which screen was
  showing

#### Scenario: Lost entitlement returns to billing from anywhere

- **WHEN** any request fails with `subscription_required`
- **THEN** the app returns to the billing step and explains why

#### Scenario: An unauthorized response clears the session

- **WHEN** any request fails as unauthorized after a session was restored
- **THEN** the stored session is cleared and the sign-in entry point is shown

### Requirement: The published conformance vectors are asserted in the app's tests

The contract artifact's vectors SHALL be checked into the test target and
asserted against the app's own decoding, so that client drift fails a test rather
than a user's transfer.

#### Scenario: Money vectors are decoded by the app's own type

- **WHEN** the test suite runs
- **THEN** each published decimal money string decodes to the expected integer
  cents through the app's money type

#### Scenario: Money is never decoded into a floating-point type

- **WHEN** the money type is inspected
- **THEN** it is backed by an integer or decimal representation and no monetary
  value passes through `Double` or `Float`

### Requirement: The update prompt is terminal and honest

The update prompt SHALL offer a route to the App Store and SHALL NOT offer a way
to continue using the app, since the server will refuse its requests.

#### Scenario: No dismissal returns to the app

- **WHEN** the update prompt is shown
- **THEN** there is no action that returns to the previous screen

#### Scenario: The prompt is distinguishable from an outage

- **WHEN** the update prompt is displayed
- **THEN** it states that the app must be updated, not that Budj is unavailable
