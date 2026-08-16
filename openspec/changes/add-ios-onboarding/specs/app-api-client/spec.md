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

#### Scenario: An unauthorized response that survives refresh clears the session

- **WHEN** any request fails as unauthorized and the refresh described below does
  not produce a usable session
- **THEN** the stored session is cleared and the sign-in entry point is shown

### Requirement: An expired access token is refreshed before the session is abandoned

An unauthorized response SHALL cause one refresh attempt before the session is
discarded, because an access token expires on a timer far shorter than the
session it belongs to, and treating expiry as rejection would sign a person out
while they are using the app.

#### Scenario: Expiry is recovered without the user noticing

- **WHEN** a request fails as unauthorized and a refresh token is held
- **THEN** the client obtains a new session and reissues the original request
  once, and the interface shows no interruption

#### Scenario: A rejected refresh token ends the session

- **WHEN** the refresh attempt is itself refused
- **THEN** the stored session is cleared and the sign-in entry point is shown,
  and no further refresh is attempted for that session

#### Scenario: Concurrent failures share one refresh

- **WHEN** several requests fail as unauthorized at the same time
- **THEN** one refresh is performed and all of them await its outcome, rather
  than each issuing its own

#### Scenario: A refresh failure is not mistaken for an outage

- **WHEN** the refresh attempt fails because the network is unreachable rather
  than because the token was refused
- **THEN** the session is retained and the failure is surfaced as a network
  failure, since a token that was never presented cannot have been rejected

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

#### Scenario: Money is read from the published string form, not a number

- **WHEN** a monetary field is decoded
- **THEN** it is read as the published decimal string and converted directly,
  rather than decoded as a number and converted afterwards, since the second
  form loses precision before the money type ever sees the value

#### Scenario: The number of decimal places is not assumed

- **WHEN** a monetary field arrives with fewer or more decimal places than two
- **THEN** it decodes to the same value it represents, because the server's
  outbound amounts are not constrained to a fixed number of places

### Requirement: The client decodes the published envelopes rather than shapes of its own

The client SHALL decode the response shapes the contract publishes — a failure
envelope, a collection envelope with its own pagination metadata, and a bare
object for a single resource — so that a shape change is a decoding failure in
one place rather than a surprise at a call site.

#### Scenario: A failed request is read from the failure envelope

- **WHEN** any request fails
- **THEN** its code and message are read from the published failure envelope,
  and a response that does not match that envelope is surfaced as a decoding
  failure rather than silently treated as success

#### Scenario: A collection carries its pagination metadata

- **WHEN** a collection endpoint responds
- **THEN** the items and the pagination metadata are both decoded, so a caller
  can tell a full page from the last one without counting

#### Scenario: Key and date conventions come from the contract

- **WHEN** any response is decoded
- **THEN** the key naming and timestamp format are those the contract publishes,
  configured once on the decoder rather than restated per model

### Requirement: Published identifiers are matched exactly, including their case

The client SHALL match every named element of the contract exactly as published —
error codes, header names, field names — and MUST NOT normalise case, trim, or
otherwise reinterpret a published identifier before comparing it.

#### Scenario: Case is part of the identifier

- **WHEN** an error code is compared against a known case
- **THEN** the comparison is exact, so a code published in one casing and
  compared in another does not match and degrades to a general server failure
  rather than silently matching

#### Scenario: Identifiers are not invented by the client

- **WHEN** a named element of the contract is required — an error code, a header
  name, a field name
- **THEN** its value comes from the published contract artifact, and the client
  defines no identifier the contract does not name

### Requirement: The server the client talks to is configurable per build

The base address of the API SHALL be a build configuration value rather than a
constant in source, so the app can be pointed at a local server, a staging
deployment, or production without a code change.

#### Scenario: A debug build can reach a local server

- **WHEN** the app is built for development
- **THEN** it can be directed at a locally running server without editing source

#### Scenario: A release build cannot be redirected

- **WHEN** the app is built for release
- **THEN** the address is fixed at build time and nothing at runtime can change
  it

### Requirement: The update prompt is terminal and honest

The update prompt SHALL offer a route to the App Store and SHALL NOT offer a way
to continue using the app, since the server will refuse its requests.

#### Scenario: No dismissal returns to the app

- **WHEN** the update prompt is shown
- **THEN** there is no action that returns to the previous screen

#### Scenario: The prompt is distinguishable from an outage

- **WHEN** the update prompt is displayed
- **THEN** it states that the app must be updated, not that Budj is unavailable
