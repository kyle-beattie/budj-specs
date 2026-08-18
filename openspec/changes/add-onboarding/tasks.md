## 1. Unblock

- [x] 1.1 ~~Determine whether App Review requires IAP.~~ **Resolved: IAP is
      required.** Section 5 is StoreKit 2 throughout; Stripe is not used. See D8.
- [x] 1.2 ~~Settle whether webhook subscription is per app or per user.~~
      Resolved: per user, after token exchange — but deferred to
      `add-rule-triggers` with the handler, since subscribing before the endpoint
      exists risks Akahu disabling it. No task here.
- [x] 1.3 ~~Decide the identity linking policy (open question 2) and record it.~~
      **Resolved: accept duplicates.** See D17. Section 4 gains no
      duplicate-detection task.

## 2. Prove the foundation

- [x] 2.1 Get `pnpm db:start` running locally and `pnpm db:reset` applying
      `00000000000001_init.sql` for the first time. Fix whatever it turns out the
      migration got wrong.

      It got two things wrong, both fatal, and neither visible without running
      it. This is the task justifying its own existence.

      **The migration was never applied — and the CLI said so quietly.**
      `supabase db reset` printed `Skipping migration 00000000000001_init.sql...
      (replace "init" with a different file name to apply this migration)` in the
      middle of otherwise successful output, and exited 0. The CLI reserves the
      name `init`. Renamed to `00000000000001_initial_schema.sql`. Every schema
      claim in this repository was resting on a file the tooling had been
      declining to read.

      **No table was reachable by the API.** Tables landed with only
      `REFERENCES,TRIGGER,TRUNCATE` for `anon`, `authenticated` and
      `service_role` — no `SELECT`, `INSERT`, `UPDATE` or `DELETE` on anything.
      RLS decides which *rows* a role may touch; it does not grant access to the
      table, and PostgREST returns `42501 permission denied for table X` before
      consulting a single policy. Every policy in this file was dead code that
      looked alive. Added an explicit, commented `grants` block, with `anon`
      absent throughout and `akahu_tokens`/`apple_grants` withheld even from
      `authenticated`, so a mistakenly added policy still would not expose them.

- [x] 2.2 Add an integration test harness that runs against the local stack,
      separate from `test/app.test.ts` and skipped when the stack is absent.
      `test/integration/harness.ts`. Also refuses to run against a non-loopback
      `SUPABASE_URL`, so it can never create users in a hosted project.
- [x] 2.3 Integration test: signing up creates exactly one `profiles` row via the
      `handle_new_user` trigger.
- [x] 2.4 Integration test: user A cannot select user B's `accounts` row through
      a user client. This is the first real proof the RLS policies work.
     
- [ ] 2.5 ~~Integration test: PostgREST returns `numeric(14,2)` as a string, as
      `CLAUDE.md` asserts.~~
      **Not currently testable, and the reason matters.** Dropping
      `accounts.balance` (3.2) removed the last `numeric` column in the schema —
      there is now no money column anywhere in `public`. The `CLAUDE.md` claim is
      not wrong, it is *vacuous*: nothing exercises it. Deferred to
      `add-rule-triggers`, which introduces payment amounts and with them the
      first numeric column this assertion can be made against. Do not delete the
      claim from `CLAUDE.md` in the meantime — it is a rule for the next column,
      not a description of a current one.

## 3. Schema

- [x] 3.1 Edit `00000000000001_init.sql` in place — permitted only because it has
      never been applied to a real database. Note in the file that migrations are
      append-only from this change onward.
- [x] 3.2 Rewrite `public.accounts`: drop `balance`, `institution`, `is_archived`
      and the `(user_id, lower(name))` unique index; add `akahu_account_id`,
      `connection_id`, `payment_from`, `payment_to`, `last_seen_at`,
      `disconnected_at`; add unique `(user_id, akahu_account_id)`. Two capability
      flags, not one — Akahu governs paying out and receiving differently.
      `connection_id` is a uuid FK to `akahu_connections(id)`, not Akahu's
      `conn_...` string, so the FK carries tenancy.
- [x] 3.3 Add `other` to the `account_type` enum as the fallback for unmapped
      Akahu types.
- [x] 3.4 Add `akahu_tokens` (`user_id`, `akahu_user_id` indexed,
      `token_ciphertext`, timestamps). Enable RLS and deliberately declare **no
      policies**; comment saying so, or someone will "fix" it.
- [x] 3.5 Add `akahu_connections` (`user_id`, `connection_id`, `name`,
      `connected_at`, `disconnected_at`) with owner RLS policies.
- [x] 3.6 Add `billing_subscriptions` (`user_id`, `original_transaction_id`
      **unique**, `product_id`, `plan_code`, `status`, `expires_at`) with owner
      select-only RLS — the server writes it, the user never does. The unique
      constraint is what stops one App Store subscription entitling two accounts.
      `status` is a new `subscription_status` enum.
- [x] 3.7 Add `device_registrations` (`user_id`, `device_id`, `apns_token`,
      `registered_at`, `revoked_at`) with owner RLS policies.
- [x] 3.8 Add `apple_grants` (`user_id`, `refresh_token_ciphertext`, timestamps).
      Enable RLS with **no policies**, same as `akahu_tokens`.
- [x] 3.8 Add `updated_at` triggers for every new table.
- [x] 3.9 Change currency defaults from GBP to NZD in the migration.
- [x] 3.10 `pnpm db:reset` then `pnpm types:generate`. Never hand-edit
      `database.types.ts`.
      Added `pnpm types:generate:local`: the existing script passes `--linked`,
      which needs a linked remote project and cannot read a local stack.

      Generating for the first time also proved `database.types.ts` **had** been
      hand-edited — it carried `InsertDto`/`UpdateDto` aliases and a simplified
      `Tables` that the generator does not emit, and they vanished on the first
      real run. Both aliases now live in `src/supabase/index.ts`, mapped onto the
      generated `TablesInsert`/`TablesUpdate`, so the generated file is once
      again purely generated and the rule in `CLAUDE.md` is true.

## 4. Identity

- [x] 4.1 Rewrite `handle_new_user` to read `full_name` then `name` from the
      identity claims, falling back to `''` — never to the email local part.
      Our own sign-up's `display_name` is still read first, ahead of the provider
      claims, so the email/password path keeps working. Empty strings are
      `nullif`-ed so a provider sending `""` does not beat a later claim.
- [x] 4.2 Integration test: a signup whose only email is a
      `@privaterelay.appleid.com` address produces an empty display name, not a
      relay fragment.
- [x] 4.3 Integration test: a signup carrying `full_name` in its metadata
      captures it.
- [x] 4.4 Write down, for the iOS team, that Apple sends the user's real name
      **only on the first authorisation ever**. It must be included on the first
      `signInWithIdToken` call or it is unrecoverable.
      `docs/ios-integration.md`, with the Swift call shape.
- [x] 4.5 Confirm the OpenAPI document contains no route accepting an Apple or
      Google *identity* token. The authorization-code route in 4.7 is the only
      permitted provider endpoint.
      Asserted rather than confirmed by eye: `test/app.test.ts` fails if any
      provider-ish path other than `/api/auth/apple/grant` appears, or if that
      route's schema ever mentions an id token.
- [x] 4.6 Apple client secret generation: an ES256 JWT signed with the key from
      Apple, regenerated before its six-month expiry. Add the key and team/key
      identifiers to `env.ts` and `.env.example`.
      Signed with `node:crypto`, not a JWT library — the construction is fifteen
      lines and this is *signing*; the dangerous half of JWT handling is
      verification, which this server does none of for Apple. Billing (5.2) adds
      a real JOSE dependency for x5c chain verification, and that is the right
      place for it. Minted per exchange with a 120-day life, so nothing is
      cached near Apple's six-month ceiling.
- [x] 4.7 `POST /api/auth/apple/grant` — accept an authorization code, exchange it
      with Apple for a refresh token, encrypt and store it using the same helper
      as the Akahu token. Reuse the deny-all RLS table pattern. A failed exchange
      is recorded and does not fail sign-in.
      "Recorded" is a `logger.warn` carrying the user id and Apple's error code,
      never the code or the token; the route answers 200 `{stored:false,reason}`.
      Guarded by `requireAuth`, so the grant binds to a user this server has
      verified rather than to whatever the body claims.
- [x] 4.8 Tell the iOS team the authorization code is **single-use and expires in
      about five minutes** — it must be posted during sign-in, in the same flow
      that captures Apple's one-shot name (4.4). Both are unrecoverable if missed.
      Same document, deliberately alongside 4.4: they are one transaction, they
      fail the same silent way, and separating them invites doing one of them.
- [x] 4.9 Tests: a user client selecting from the Apple grant table gets zero
      rows; the stored token is ciphertext; re-submitting a code replaces rather
      than duplicates.
      `test/integration/apple-grant.test.ts` — owner, stranger and anon all see
      zero rows while the service role sees the row, so the assertion cannot
      pass by the row simply being absent.

## 5. Billing

- [x] 5.1 Add the App Store Connect API credentials to `env.ts` and
      `.env.example`: issuer id, key id, the `.p8` private key, bundle id and app
      Apple id. The suite fails at import without the latter file.
      Also `APP_STORE_ENVIRONMENT` (Sandbox|Production): the two sign different
      chains and issue different transaction ids, and accepting both would let a
      sandbox purchase entitle a production account. Kept clearly separate from
      the Sign in with Apple key — they are different credentials and confusing
      them is a half-day of `invalid_client`.
- [x] 5.2 JWS verification helper: validate the x5c certificate chain to Apple's
      root and extract the payload. Used by both transaction submission and
      server notifications, so build it once with its own unit tests — a
      permissive implementation here grants free subscriptions to anyone.
      `apple-jws.ts`, with Apple Root CA - G3 pinned as bytes in
      `apple-root-ca.ts` (fingerprint verified out of band and re-asserted in
      the test, so an edit fails the build).

      **The anchor is compared with `Buffer.equals`, not by name.** The fixtures
      include a self-signed certificate whose subject and issuer are byte-for-
      byte Apple's — one `openssl` command — and a test proves the verifier
      rejects it. Name-matching is the plausible-looking way to build this and
      it is worthless.

      20 tests: impostor root, missing intermediate, expired and not-yet-valid
      certificates, `alg: none`, an HS256 swap, a tampered payload, a payload
      signed by a non-leaf key, and an over-long chain. The trust anchor is
      injectable **only** so the accept path is testable at all — nobody can
      mint a chain under Apple's real root — and a test asserts the default
      anchor rejects the fixture chain.
- [x] 5.3 Define the plan catalogue and entitlement resolver in code, mapping
      `plan_code` to App Store product identifiers. No table.
      `plans.ts`. `planByProductId` returns undefined for an unrecognised
      product rather than guessing — that is what a notification for a product
      published after this build looks like, and the caller decides.
- [x] 5.4 `billing.repository.ts` — read and write the cached entitlement row.
      Unique constraint on `original_transaction_id`.
      Adds `findByOriginalTransactionId`: Apple knows about a purchase and
      nothing about our users, so that column is the only join a notification
      has. Writes require a service-role client — the table is select-only for
      its owner.
- [x] 5.5 App Store Server Notifications V2 endpoint, before anything that grants
      entitlement, so state can be observed before it can be made. Runs as
      service role; document why `requireAuth` does not apply.
      `POST /api/billing/apple/notifications`. `test/app.test.ts` asserts an
      anonymous caller reaches *validation* rather than a guard — a 401 there
      would mean someone added `requireAuth`, and Apple's deliveries would fail
      silently until entitlement drifted far enough to notice. An unverifiable
      payload is 400, not 500 and not 200: a retry cannot make it verify.
      Recorded as the third service-role exception in `CLAUDE.md`.
- [x] 5.6 Handle `SUBSCRIBED`, `DID_RENEW`, `DID_CHANGE_RENEWAL_STATUS`,
      `EXPIRED`, `DID_FAIL_TO_RENEW`, `GRACE_PERIOD_EXPIRED` and `REFUND`
      idempotently. Every notification ending entitlement runs the same
      revocation path (D9).
      `entitlement.ts` holds the mapping, pure and no-I/O like `rules.engine.ts`,
      so the whole table tests directly.

      Two readings had to be settled. `DID_CHANGE_RENEWAL_STATUS` does **not**
      end entitlement — the user has paid through the current period and
      `EXPIRED` ends it later; revoking on it cuts off someone still owed
      service, and it is tempting because the notification reads like a
      cancellation. `DID_FAIL_TO_RENEW` with subtype `GRACE_PERIOD` keeps
      entitlement, otherwise `GRACE_PERIOD_EXPIRED` would have nothing left to
      mean.

      **Idempotency needed schema.** Deterministic upserts survive replay but
      not reordering, and `updated_at` cannot stand in — it records when *we*
      wrote, not when Apple signed, so a notification signed at 10:02 and
      delivered at 10:06 looks older than one signed at 10:00 and written at
      10:05. Migration `00000000000002` (the first append-only one) adds
      `last_notification_uuid` for replay and `last_notification_at` for
      ordering, compared on Apple's `signedDate`.

      **Revocation is half-built, deliberately.** `LocalBankAccessRevoker` marks
      connections and accounts disconnected but does **not** delete the stored
      Akahu token, because calling Akahu's `DELETE /token` needs the client that
      arrives in section 6. Deleting our copy first would destroy the only means
      of ever making that call — Akahu would keep the connection alive and keep
      billing, with nothing left to revoke it with. Keeping the row is
      recoverable; discarding it is not. The ordering for section 6 is fixed:
      call Akahu, then delete. Until then a revoked user costs a fee, which is
      free right now because nobody has connected a bank.
- [x] 5.7 `POST /api/billing/transaction` — accept a signed StoreKit transaction,
      verify it, bind it to the authenticated user. Reject one already bound to a
      different user.
      The user comes from the verified JWT and never from the payload — a test
      submits a transaction naming a different user and asserts it changes
      nothing. Also refuses a foreign `bundleId`, a mismatched App Store
      environment (sandbox and production issue overlapping ids, so accepting
      both is a free subscription for anyone with Xcode), an unknown product,
      and will not resurrect entitlement from a refunded transaction.
      Idempotent, because StoreKit replays unfinished transactions on launch —
      resubmission must be boring, not an error. The write runs as service role
      and is now the fourth documented exception in `CLAUDE.md`.
- [x] 5.8 `GET /api/billing/plans` and `GET /api/billing/subscription`. No
      cancellation route: the server cannot cancel, and must not pretend to.
      Both `requireAuth` only. `subscription` reports `active` as a computed
      field rather than echoing `status`, so a lapsed-but-still-`active` row
      reads as inactive to the client too.
- [x] 5.9 `requireSubscription` hook returning 402 `subscription_required`; apply
      to bank connections and rules, and deliberately not to onboarding status.
      Applied to `rules`. Bank connections and onboarding status do not exist
      yet; both are wired when their modules land in sections 6 and 9.

      `isCurrentlyEntitled` refuses a subscription whose `expires_at` has passed
      even when its status still reads `active` — the spec forbids inferring
      continued entitlement from the absence of a notification, and a missed
      `EXPIRED` would otherwise serve someone free indefinitely. `grace_period`
      is the deliberate exception: the expiry has by definition passed while
      Apple retries payment, and enforcing it there cuts off exactly the people
      Apple is still trying to bill.
- [x] 5.10 Tests: unverifiable notification rejected with no write; a transaction
      bound to another user refused; resubmission idempotent; unsubscribed caller
      gets 402 from a gated route and 200 from status; the OpenAPI document has
      no cancellation route.
      All five pass. The 402 cases are end-to-end in
      `test/integration/billing.gate.test.ts` — a real user, a real Supabase
      token, real RLS — including that 401 and 402 stay distinguishable, that
      the catalogue and subscription reads stay reachable unpaid, and that a
      silently lapsed subscription is refused.
- [ ] 5.11 Sandbox testing with a StoreKit configuration file, including a refund
      and an expiry. These are the paths that revoke bank access and they cannot
      be exercised in production.
      **Written, not run — and it needs a person.** `contract/Budj.storekit`
      plus `docs/storekit-sandbox-testing.md` with the eight cases worth the
      trouble. Executing it needs Xcode, a device and an App Store Connect
      sandbox account, none of which the suite can reach, so this stays open
      until someone runs it.

      Worth knowing before trying: transactions from Xcode's *local* StoreKit
      testing are signed by a local test certificate, not by Apple, so chain
      verification rejects them by design. Only the App Store Connect sandbox
      exercises the real path. The doc says so rather than letting someone
      discover it as a bug.

## 6. Bank connections

- [x] 6.1 Add the Akahu environment variables and ~~`AKAHU_TOKEN_ENC_KEY`~~
      `TOKEN_ENC_KEY` to `env.ts` and `.env.example`.
      Done. `AKAHU_APP_TOKEN`, `AKAHU_APP_SECRET` and `AKAHU_REDIRECT_URI` land
      here with the client that uses them. The redirect URI must match what is
      registered with Akahu byte-for-byte, trailing slash included — a mismatch
      fails the *code exchange* rather than the redirect, so it presents as a
      token problem.

      **Renamed to `TOKEN_ENC_KEY`.** It encrypts Apple's refresh token too, and
      a key named `AKAHU_` that also protects Apple credentials is precisely the
      name that causes an incident during rotation — someone rotates "the Akahu
      key", not realising they have orphaned every Apple grant as well. The
      runbook (11.5) is only useful if the name is honest. D4 updated.
- [x] 6.2 Envelope encryption helper with a key-version prefix on the ciphertext,
      so rotation is possible later. Unit tests for round trip and for rejecting
      an unknown key version.
      `src/lib/token-crypto.ts`, AES-256-GCM, `v1:<base64url(iv|tag|ct)>`.
      The keyring takes a comma-separated list with the newest first, so
      rotation is "prepend the new key" rather than a migration: 18 unit tests
      cover round trip, unknown version, a tampered ciphertext, a ciphertext
      relabelled with another version, and that two encryptions of the same
      plaintext never match.
- [x] 6.3 `getAkahuToken(userId)` — the only non-admin service-role read in the
      codebase. It returns a credential and never a row. Comment it as the
      documented exception.
      `token.repository.ts`. Returns the decrypted string or null, deliberately
      not the row: no timestamps, no Akahu user id, nothing that invites
      treating it as ordinary data access. `forget()` carries the ordering
      warning that matters — see the revoker below.
- [x] 6.4 Integration test: a user client selecting from `akahu_tokens` gets zero
      rows. This is the load-bearing assertion of the whole custody design.
      Done early, with the table, in `test/integration/rls.test.ts` — an
      unpoliced table shipping unproven is the worst failure mode available.
      Covers the owner, a stranger and anon, and shows the row does exist to the
      service role. Repeated in `test/integration/akahu-custody.test.ts` against
      a token stored through the real repository.
- [x] 6.5 Akahu client wrapper: ~~authorisation URL construction~~, code exchange,
      `GET /accounts`, ~~`GET /connections`~~, `DELETE /token`.
      Two corrections, both from reading Akahu's API reference rather than
      assuming it.

      **The server does not construct the authorisation URL.** It posts a
      *pushed authorisation request* to `POST /v1/par` and Akahu returns the
      `authorisation_url`. This is not a style preference: with the inline
      `oauth.akahu.nz` URL the granular scopes come from static app
      configuration in Akahu's dashboard and the `scope` parameter carries only
      `ENDURING_CONSENT`, so D7's "exactly these five scopes" would be neither
      true nor assertable, and the scope set would live in a dashboard instead
      of this repository. D7 updated.

      **`GET /v1/connections` is not the user's connections.** It is the
      catalogue of institutions Akahu supports, authenticated with app
      credentials, and says nothing about any particular person. A user's
      connections come from the nested `connection` object on each account, so
      the sync needs one call rather than two. Not implemented, because nothing
      needs an institution directory yet.

      Also: `balance` is deliberately absent from the account schema, so Zod
      strips it. Akahu may return one; it cannot reach the projection even by
      accident.
- [x] 6.6 `POST /api/bank-connections/authorise` — issue an authorisation URL with
      a single-use expiring `state` bound to the user, and the exact five scopes
      from D7. Assert in a test that `payments` and `accounts:balance` are absent.
      State is 256 bits of entropy in `akahu_auth_states` (migration
      `00000000000003`), RLS enabled with no policies — it is a capability, not
      user data, and a test asserts no user client can read it.
- [x] 6.7 Redirect handler: validate and consume `state`, exchange the code,
      encrypt and store the token, record the connection, sync accounts. Ensure
      no code path writes a plaintext token, even transiently.

      **Built as an authenticated `POST /callback`, not an unauthenticated
      `GET`.** The obvious shape identifies the user *solely* from the state,
      which makes that state the only thing between a leaked redirect URL and a
      stranger's bank being attached to your account. The client is a native
      app: it intercepts the redirect itself and posts the code with its own
      bearer token, so the exchange is bound twice — by a JWT this server
      verified and by a state it issued. A web client would need the `GET` form,
      and that is the moment to think hard about it. `test/app.test.ts` fails if
      a `GET` callback appears.

      Ordering: the state is consumed **before** the code is exchanged.
      Reversing it burns a one-shot code on a replayed request and obtains a
      token before establishing whose it is. A test asserts no exchange happens
      when the state is refused.

      Consuming is a single conditional update (`.is('consumed_at', null)`), not
      a read-then-write, so two simultaneous redirects race in Postgres and
      exactly one wins.
- [x] 6.8 Account projection sync: upsert on `(user_id, akahu_account_id)`, map
      Akahu types with an `other` fallback, set `payment_from` from the
      `PAYMENT_FROM` attribute and `payment_to` from ~~BECS eligibility~~ the
      `PAYMENT_TO` attribute, set `last_seen_at`, and mark accounts absent from
      the response as disconnected rather than deleting them.

      Mapping is pure and separately tested. `FOREIGN`, `TAX` and `REWARDS` map
      to `other` deliberately rather than being forced into a local type — a
      foreign-currency account is not a cheque account and a rewards balance is
      not money, and `other` says "we do not model this", which is true.

      **One guard beyond the task:** an empty account list never disconnects
      anything. An empty response is ambiguous — "removed everything" or "Akahu
      had a bad minute" — and treating the second as the first marks every
      account disconnected and, once `add-rule-triggers` lands, silently stops
      every rule the user has. The cost is a stale row for a user who genuinely
      removes their last account; the cost the other way is much worse.
- [x] 6.9 Enforce the plan's connection limit before starting authorisation.
      Before, specifically: sending someone through their bank's authorisation
      screen only to refuse them on the way back would be hostile. 403
      `PLAN_LIMIT_EXCEEDED`, distinct from 402 — they have paid, so the app
      needs to offer an upgrade rather than a subscription.
- [x] 6.10 `GET /api/bank-connections` and a revoke route that marks the
      connection and its accounts disconnected.
      Plus `POST /sync` to re-run the projection. Revoking one connection
      deliberately does **not** revoke the Akahu token: that token covers every
      connection the user has, so revoking it would silently disconnect their
      other banks. Losing entitlement revokes the token (D9); removing one bank
      does not.
- [x] 6.11 Tests: reused `state` refused; duplicate account names both stored;
      unmapped account type does not abort the sync; revoking marks rather than
      deletes.
      All four, plus: a state issued to another user refused, an expired state
      refused, no code exchanged when the state is refused, an account absent
      from a later sync marked rather than deleted, a reappearing account
      reconnected, and one user unable to disconnect another's connection.

      These tests also caught a real bug. `grant ... on all tables in schema
      public` in the initial migration is a **snapshot**, not a standing rule,
      so `akahu_auth_states` was created with no privileges for `service_role`
      and every insert failed `42501` — which reads like an RLS problem and is
      not one. Every future migration must grant on its own tables; recorded in
      `CLAUDE.md`.

## 7. Accounts rewrite

- [x] 7.1 Delete the create, update and delete routes, their DTOs, and their
      repository methods.
- [x] 7.2 Rewrite `accountSchema` for the projection: no balance, no account
      number, plus `akahuAccountId`, `connectionId`, ~~`paymentEligible`~~,
      `disconnectedAt`.
      **Two flags, not one:** shipped as `paymentFrom` + `paymentTo`, following
      D6 and 3.2. The single `paymentEligible` in this task's wording predates
      that decision and would have collapsed a distinction the rule editor needs
      — a credit card can trigger a rule and can never receive money.
      Also added `lastSeenAt` and a `connectionId` filter on the list query.
- [x] 7.3 Update `test/app.test.ts` — the guarded-route assertions for the
      removed verbs must become assertions that the routes are gone. They now
      assert 404, not 401: there is no route to authenticate to.
- [x] 7.4 Test: a response for an account contains no balance field.
      Asserted against the **OpenAPI document** rather than a response body — the
      iOS client is generated from it, so a `balance` property there would
      produce a field the server can never populate. Needs no database, so
      unlike the integration tests this one has actually run.

      Section 7 landed here rather than in its own PR because 3.2 makes the old
      `accounts` module fail to typecheck the moment the types are regenerated.
      Splitting them would mean shipping a red tree.

## 8. Devices

- [x] 8.1 `POST /api/devices` — register device id and APNs token. Upsert on
      `(user_id, device_id)`. No key material: the schema carries an identifier
      and a token, nothing else.
      Re-registering also clears `revoked_at`: someone who revoked a device and
      then reinstalled has un-revoked it. `test/app.test.ts` asserts the whole
      OpenAPI document mentions no `publicKey`, attestation or Secure Enclave
      field anywhere — D10 was a product decision, not an omission, and this is
      the assertion that keeps it one.
- [x] 8.2 `GET /api/devices` and `DELETE /api/devices/:deviceId`, marking revoked
      rather than deleting.
      The APNs token is deliberately absent from every response: the client sent
      it and has no use for it back, and echoing a delivery credential to
      anyone who can list devices is free risk.
- [x] 8.3 Tests: two devices coexist; re-registering one device replaces its
      token; revoking someone else's device returns 404.
      All three, plus: the owner's registration is untouched by the failed
      revocation, a revoked device stops being listed, and a payload carrying
      key material stores none of it.

## 9. Onboarding status

- [x] 9.1 `GET /api/onboarding/status` deriving the step in the order billing,
      bank, ready, with push reported as advisory.
      Bank completion keys on **token presence**, not connection rows. A user
      whose authorisation succeeded but whose sync returned nothing has a token
      and no connections; keying on connections would send them back through
      their bank, when a re-sync is what they need. `hasToken` returns a boolean
      rather than a row or a credential, so it sits inside the existing D5
      exception rather than widening it.

      The billing check reuses `isCurrentlyEntitled`, so the reported step and
      the 402 gate can never disagree — including for a silently lapsed
      subscription.
- [x] 9.2 Guard with `requireAuth` but deliberately **not** with
      `requireSubscription`.
      Asserted in `test/integration/billing.gate.test.ts` alongside the converse
      — that bank connections and devices *are* gated. Asserting both directions
      is what stops the gate drifting onto, or off, the wrong routes.
- [x] 9.3 Tests: one per step transition; a complete user with no APNs token
      still reports `ready`; an anonymous caller gets 401.
      The transition test is the one that earns its place: it grants entitlement
      through the billing repository and asserts the *very next* status request
      reports `bank`, with nothing having advanced the user. That is D1 working,
      rather than D1 asserted.
- [x] 9.4 Assert no generated type carries an onboarding step column.
      A test over `database.types.ts` rather than an inspection: six candidate
      column names, plus the word "onboarding" appearing nowhere in the
      generated types at all.

## 10. Client contract and version gating

- [x] 10.1 `requireSupportedClient` hook reading the build identifier from every
      request. A missing identifier is unsupported, not exempt — a client that
      cannot be identified cannot be gated. Exempt the App Store notification and
      Akahu webhook routes, which are not client requests.
      `src/plugins/client-version.ts`, header `x-client-build`, integer builds
      only — comparing semver during an incident is a class of bug this does not
      need. Exemptions are prefix-matched with an equality guard, so
      `/healthzz` and `/api/billing/apple/notifications-evil` do not inherit
      them; both are tested.

      Registered **before** auth: an unsupported build should be told to update
      rather than to sign in.
- [x] 10.2 Environment-driven minimum supported build and a blocked-build range
      for money-moving operations, kept separate. Environment, not a table: this
      must change during an incident without a migration.
      `MIN_SUPPORTED_BUILD` and `BLOCKED_MONEY_BUILDS` (`412-418`).

      An unset minimum disables the gate, which is right for local development
      and wrong for production — so `env.ts` **refuses to boot in production
      without it**. Otherwise "forgot to configure it" silently means "no
      gating", and the gap is invisible until the day a bad build needs
      stopping.
- [x] 10.3 A distinguishable error code for "update required", separate from
      authentication and subscription failures, so the app can prompt correctly.
      `426 CLIENT_UPDATE_REQUIRED`, and `409 CLIENT_BUILD_BLOCKED` for the money
      gate. Four distinct outcomes now, each leading somewhere different in the
      app: 401 sign in, 402 subscribe, 403 upgrade plan, 426 update the app.
- [x] 10.4 Money-movement gate applied where payments are initiated. Nothing
      initiates payments in this change, so wire the hook and prove it refuses;
      `add-rule-triggers` inherits a working gate.
      `requireMoneyMovementAllowed` is decorated and proven against a route
      declared in the test: a blocked build gets 409 on the payment route and
      200 on everything else in the same request sequence. An unidentifiable
      client is not trusted with money either.
- [x] 10.5 Emit `contract/openapi.json` from the generated document as a tagged
      release artifact.
      `pnpm contract:emit` builds the real app and writes the document its own
      routes produce, so a route that does not exist cannot appear in it.
      `.github/workflows/release.yml` attaches it on `v*` tags.

      **Not committed** — it is ~9,700 lines and would swamp the diff of every
      schema change, which is the opposite of reviewable. `money-vectors.json`
      *is* committed: small, hand-curated, and the suite asserts the server
      against it.
- [x] 10.6 Write `contract/money-vectors.json`: decimal strings to integer cents
      including rounding cases, and assert the server's own implementation
      against them in CI.
      `src/lib/money.ts` parses by string manipulation and rejects three decimal
      places rather than rounding — a rounding rule is a decision about
      someone's money, and making it silently would hide the bug that produced
      the third digit.

      The vectors include five values where `parseFloat(s) * 100` lands just
      under the integer (`0.29`, `4.35`, `1.15`, `9.95`, `1.13`), so a client
      that truncates loses a cent on some inputs and not others. A test asserts
      those cases genuinely still trap, so the set cannot quietly stop being
      one.

      *Correction while writing this:* an earlier draft claimed `8.16` was such
      a case. It is not — `8.16 * 100` is exactly `816`. The vector is kept,
      relabelled as a near miss, so the set is not all traps.

      CI did not exist, so `.github/workflows/ci.yml` was added. It runs
      typecheck and tests, and fails if regenerating the vectors produces a diff
      — a committed vector file that disagrees with the server would be
      believed.
- [x] 10.7 Tests: a below-minimum build refused with the update code; a request
      with no build identifier refused; a blocked build refused only for
      money-moving routes; webhooks unaffected.
      42 tests. The decision logic is pure and tested exhaustively; the HTTP
      behaviour is tested against a small Fastify app built in the test, which
      is what makes a *configured* gate testable at all — `config` is read once
      at import, so the real app cannot be reconfigured per test.

- [x] 4.13 ~~Serve the address-confirmation bridge.~~ **Done.**
      `GET /auth/confirm`, mounted outside `/api` because a browser reaches it
      rather than the app, in `modules/auth/confirm.routes.ts`. One handler
      replying `text/html`, no new dependency.

      **It is a page, not a 302**, and the end-to-end run proved why: Supabase
      answers the verify link with `303 → …/auth/confirm#access_token=…`. The
      session is in the *fragment*, which is never sent to the server, so a
      redirect handler would have nothing to forward. The page reads
      `location.hash` in the browser and hands it to `budj://auth/confirm`
      unchanged, strips it from the address bar and the history entry first, and
      leaves an "Open Budj" button for the embedded browsers that block the
      automatic hop.

      **Three things this turned up that the plan did not have:**

      1. **The client-build gate had to exempt it.** A mail client sends no
         `X-Client-Build`, and a missing header is treated as unsupported by
         design — so without the exemption the bridge answered 426 and told
         someone to update an app they were in the middle of signing into.
      2. **CSP had to be set per route.** Helmet's global policy is disabled
         outside production, so the inline script the page exists to run would
         have worked in development and been refused by `script-src 'self'` in
         production. The route now carries its own policy with a sha256 of the
         served script, computed from the same string that is embedded, so the
         two cannot drift.
      3. **Password reset could not share the redirect.** `config.auth.redirectUrl`
         fed both `signUp` and `resetPasswordForEmail`. A recovery link *also*
         returns a session, so pointing that one value at the bridge would have
         signed someone in and told them their email was confirmed when what they
         asked for was a new password. Sign-up now uses a separate
         `config.auth.confirmUrl`, defaulting to this server's own route so the
         flow works with nothing configured; reset is untouched.

      **Turning `enable_confirmations` on broke 22 integration tests**, and the
      fix belonged in the harness rather than in the setting. `signUpTestUser`
      asserted a session came back and named the setting in its own error
      message, so the whole integration suite silently depended on local being
      configured unlike production. It now signs up for real — `handle_new_user`
      still fires on a genuine insert, which is what `profiles.test.ts` is there
      to check — and, when no session comes back, confirms the address through
      the admin API and signs in. Works either way round now, so the setting is
      free to match production.

      `enable_confirmations` is now `true` in `supabase/config.toml`, and the
      bridge is in `additional_redirect_urls` under both spellings of localhost —
      Supabase matches redirect targets exactly and silently falls back to
      `site_url` otherwise. Verified end to end against local Supabase: sign-up
      returns `{session: null, confirmationRequired: true}`, the email links to
      `…/auth/v1/verify?…&redirect_to=http://localhost:3000/auth/confirm`, that
      redirects to the bridge with the session in the fragment, and the refresh
      token from it exchanges at `/api/auth/refresh` for a session with
      `emailConfirmed: true`. Nine tests in `test/auth.confirm.test.ts`.

## 11. Wiring and documentation

- [x] 11.1 Register `billing`, `bank-connections`, `devices` and `onboarding` in
      `src/modules/index.ts`.
      Done as each module landed. All eight are registered; the generated
      OpenAPI document contains 26 paths across them.
- [x] 11.2 Update `test/app.test.ts` for the new mount points and guards.
      Also added the assertion the named list cannot make: a **completeness
      check** that walks every path in the generated OpenAPI document and
      requires each to be either on an explicit public list — with a written
      reason — or rejecting anonymous callers. A route added later is otherwise
      simply absent from the named list and nobody notices. It asserts it
      checked more than twenty routes, so an empty document cannot satisfy it
      vacuously, and a second test fails if a route on the public list has since
      been guarded.
- [x] 11.3 Update `CLAUDE.md`: the ~~two~~ **five** service-role exceptions (D5),
      accounts as a projection rather than a user-owned table (D6), the client
      version gate (D15), the new modules, the new environment variables, and
      the fact that an integration suite now exists.
      Written incrementally as each piece landed, then audited as a whole here.
      The exception list grew from two to five — Apple grants, the notification
      handler and purchase submission joined the two the design anticipated —
      and it is now written as a **closed list** where a sixth entry is the
      thing to review.

      Added in the audit: a module table with the guard each one applies,
      corrected "adding a module" steps (the grant, and `types:generate:local`),
      an environment section pointing at the runbook, the pure-function pattern
      now used in five places, and an explicit note that **every external
      provider is stubbed and nothing has ever called the real Akahu API**.
- [x] 11.4 Update `.env.example` with every new variable and confirm
      `pnpm test` passes from a clean checkout with placeholders only.
      **Confirmed by actually doing it**, not by inspection: a fresh clone with
      `cp .env.example .env` and nothing else gives a clean typecheck and
      310 passing / 84 skipped — the skips being the integration suite
      correctly detecting no local stack. CI now runs exactly that on every
      pull request, so the claim cannot quietly stop being true.
- [x] 11.5 Write the key runbook: ~~`AKAHU_TOKEN_ENC_KEY`~~ `TOKEN_ENC_KEY` —
      where it lives, what breaks if it is lost, how to rotate using the version
      prefix — and the Apple signing key, which expires every six months and
      silently breaks the code exchange when it does. Closes open question 4.
      `docs/key-runbook.md`, covering four secrets rather than two: the two
      named here plus the App Store Connect key and the Supabase service role
      key, because the first two are easy to confuse and the last is the one
      with unlimited blast radius.

      The part worth reading twice is that losing `TOKEN_ENC_KEY` is
      **unrecoverable, not merely disruptive**. Bank reconnection is
      inconvenient; the Apple grants cannot be recaptured at all, because the
      authorization code is single-use and issued only at sign-in. Until each
      affected user signs in again, their account cannot be properly deleted.

      Also documents the failure mode nobody would otherwise find: an expired
      Apple key produces no outage, because a failed exchange is deliberately
      swallowed so it cannot break sign-in. The symptom is a `logger.warn` per
      sign-up and a growing population of unrevocable users. Set a calendar
      reminder at five months.
- [x] 11.6 Tell the iOS repo what to pin: the contract tag, the vectors to run in
      its own CI, and that every request must carry a build identifier.
      `docs/ios-integration.md`, which also carries the two one-shot Apple
      values, the bank connection flow, and a table of the four refusals that
      lead to four different screens — collapsing any two sends someone
      somewhere that cannot fix their problem.
- [x] 11.7 `pnpm typecheck && pnpm test` clean, and the OpenAPI document
      generates.
      Typecheck clean, 396 tests passing with the integration suite running
      against a local stack, and `pnpm contract:emit` produces a 26-path
      document. CI enforces all three plus vector freshness.
