# 3. User lifecycle: administration and Keycloak consistency

- **Status:** Accepted
- **Date:** 2026-11-03 (proposed and accepted)

## Context

User data is split (ADR-0002): Keycloak holds the account (login email,
password, profile) and `service-identity` holds the CompanyCentral user
(ID, memberships, later invitations), linked by the Keycloak subject (`sub`).
Administrators must be able to edit and delete users without leaving the two
out of sync, without breaking history in other services (which store user
IDs, for example who released a work order) and without a deleted user keeping
access through a still-valid token.

Two systems cannot be updated in one transaction, and Keycloak also has its
own admin console, where accounts can be changed without CompanyCentral
knowing.

## Options

**A. Keycloak console as the admin tool, identity follows.** Identity learns
about changes from Keycloak events. Keycloak has no standard way to publish
them; it needs a custom Java event-listener extension (new technology,
deployed into Keycloak) and still misses changes made while it was down.

**B. CompanyCentral admin API as the admin tool; identity is the only writer
to Keycloak.** Identity changes Keycloak through its Admin REST API with a
dedicated service account and applies its own change in the same operation,
with retries for the Keycloak side. Changes made in the Keycloak console are
caught by a reconciliation job.

**C. Reconciliation only.** Admins use the Keycloak console; identity
periodically copies its state. Simple, but deletions are seen late and the
gap is a security window.

## Decision

**Option B**, with reconciliation as a safety net.

- **Who:** a `platform-admin` Keycloak realm role may list, edit and delete any
  account. The gateway passes the user's platform roles in the signed context;
  identity enforces them. Company-level administration (removing a member
  from a company) is a membership change, not an account change, and comes
  with memberships and Access.
- **Edit:** identity updates Keycloak first (it owns the login email and
  rejects duplicates), then its own row. If the second step fails, the profile
  sync at the next sign-in and the reconciliation job repair it.
- **Delete = anonymize:** in one database transaction identity marks the user
  `deleted`, erases email and names (the ID stays, so references in other
  services remain valid and show a deleted user), deactivates all memberships,
  writes `identity.user.deleted` to its outbox and queues a "delete account in
  Keycloak" task. A worker runs the task with retries and exponential
  backoff (a missing account counts as success). After 50 failed attempts
  (about eight hours) the task is parked and logged as an error for an
  operator, so a permanent failure cannot retry forever; the local user stays
  anonymized either way. The email is free for a new account at once.
- **Access ends at once:** the gateway resolves the token subject to the
  CompanyCentral user through identity on every request (short cache) and
  rejects deleted users, even before the Keycloak token expires.
- **Keycloak console changes:** the reconciliation job compares accounts and
  users. An account that Keycloak reports as gone (a per-user `404`, never an
  empty list) is anonymized as above; changed emails and names are copied.
  The number of deletions per run is capped, so a misconfiguration cannot wipe
  users. Sign-in, admin edits and reconciliation also repair an email
  conflict left by a console deletion instead of failing.
- **Provider independence:** the Keycloak Admin API is used only by identity,
  behind one adapter module (`src/idp/`). This narrows ADR-0002's "standard
  OIDC only" rule: the gateway and other services still use standard OIDC
  only.
- **Events:** identity publishes through a transactional outbox (architecture
  §5) using `@confluentinc/kafka-javascript`, Confluent's maintained client
  (KafkaJS has had no release since 2023).

## Consequences

- One path for user changes, with no lost deletions: every step after the
  database commit is retried from a durable task. A task that keeps failing
  is parked for an operator rather than retried forever, so a Keycloak
  account can outlive its anonymized user until someone acts on the alert.
- Identity needs a Keycloak service account with `manage-users`; its secret is
  a Kubernetes Secret like the database passwords.
- Every request costs one cached lookup in identity; the gateway depends on
  identity being available (it already does for membership checks).
- Changes in the Keycloak console are repaired within one reconciliation
  interval, not immediately; the admin UI is the supported tool.
- Anonymized users stay as rows without personal data; a hard delete is not
  offered.
