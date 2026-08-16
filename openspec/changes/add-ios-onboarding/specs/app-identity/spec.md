## ADDED Requirements

### Requirement: The identity token goes to Supabase and never to this project's server

The app SHALL exchange the provider's identity token with Supabase directly to
obtain a session. It MUST NOT send an identity token to the Budj server on any
route.

#### Scenario: Sign-in produces an ordinary session

- **WHEN** a user completes Sign in with Apple
- **THEN** the identity token is exchanged with Supabase and the resulting access
  token is used for subsequent Budj requests

#### Scenario: No request body carries an identity token

- **WHEN** the app's outbound requests to the Budj server are inspected
- **THEN** none contains an Apple or Google identity token

### Requirement: Apple's authorization code is captured at sign-in and posted to the server

The app SHALL send Apple's authorization code — a distinct artifact from the
identity token — to the Budj server as part of sign-in. The code is single-use and
short-lived, so there is no later opportunity to obtain it.

#### Scenario: The code is posted after the session exists

- **WHEN** Sign in with Apple completes and a Supabase session has been
  established
- **THEN** the authorization code is posted to the server on the authenticated
  Apple grant route

#### Scenario: The code and the identity token are not confused

- **WHEN** the sign-in path is tested
- **THEN** the value posted to the server is the credential's authorization code
  and the value sent to Supabase is the credential's identity token

#### Scenario: A failed exchange does not fail sign-in

- **WHEN** the server rejects or fails to exchange the authorization code
- **THEN** the failure is recorded and the user proceeds into onboarding signed in

### Requirement: The provider-supplied name is forwarded on first authorisation only

Apple supplies the user's name only on the first authorisation. The app SHALL
forward it when present and SHALL NOT derive a display name from an email address
when it is absent.

#### Scenario: Name is forwarded when Apple provides it

- **WHEN** a first-time Sign in with Apple returns name components
- **THEN** they are included in the Supabase sign-in so the profile can be seeded

#### Scenario: A private relay address is never used as a name

- **WHEN** the credential carries no name and the only email is a
  `privaterelay.appleid.com` address
- **THEN** no display name is derived from it by the app

### Requirement: Email and password sign-in remains available

The app SHALL offer email and password sign-in and registration as a fallback for
users who do not use a provider.

#### Scenario: Password sign-in produces the same session shape

- **WHEN** a user signs in with email and password
- **THEN** the resulting session is used identically to a provider-issued one and
  routing proceeds on onboarding status

### Requirement: The session is stored in the Keychain, never in preferences

The refresh token SHALL be stored in the Keychain with device-only accessibility,
and optionally behind a biometric access control. It MUST NOT be written to user
defaults or any unprotected store.

#### Scenario: No token in user defaults

- **WHEN** the app's user defaults are inspected
- **THEN** they contain no access token, refresh token, or password

#### Scenario: Declining biometrics still persists the session

- **WHEN** a user declines biometric unlock
- **THEN** the session is stored without the biometric access control and the app
  still resumes without signing in again

### Requirement: Signing out clears local session material

Signing out SHALL remove the stored session and return the app to the sign-in
entry point.

#### Scenario: Sign-out removes the Keychain item

- **WHEN** a user signs out
- **THEN** the stored session is deleted and a relaunch shows sign-in
