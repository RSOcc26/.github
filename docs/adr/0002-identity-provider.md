# 2. Identity provider: custom or external OIDC

- **Status:** Accepted
- **Date:** 2026-10-27 (proposed 2026-10-23)

## Context

Users sign in once and then work in one or more companies. The RSO course
awards extra points for OAuth2/OIDC-based IAM. Exactly one authentication
approach will be built.

## Options

**A. Custom identity in `service-identity`.** Identity stores password hashes
(Argon2id), manages sessions and issues signed tokens (JWT, keys as JWKS). The
gateway verifies them.

- Full control over membership and invitations in one place; no extra infrastructure.
- We own security-critical code: password storage, reset, lockout, key rotation.
- Weaker claim to the OAuth2/OIDC IAM criterion.

**B. External OIDC provider (for example Keycloak).** The provider handles
login, passwords, MFA and tokens; the gateway validates OIDC tokens.
`service-identity` keeps companies, memberships and invitations and maps the
OIDC subject (`sub`) to users.

- Standard flows, MFA and account management; clearly meets the IAM criterion.
- One more stateful component (Keycloak needs its own database).
- User data is split between the provider and Identity.

## Decision

**Option B: an external OIDC provider, Keycloak.** CompanyCentral does not
build its own authentication.

- Keycloak handles login, passwords, MFA, sessions and token issuance.
  The web app signs users in with the authorization code flow with PKCE.
- The gateway validates Keycloak's tokens (signature against the
  provider's JWKS, issuer, audience, expiry) and resolves the OIDC subject
  (`sub`) to the CompanyCentral user ID. Membership and tenant checks are
  unchanged and stay in the gateway and `service-identity`.
- `service-identity` stores no passwords or credentials. It keeps users,
  companies, memberships and invitations, and maps each user to its external
  subject reference (issuer + `sub`).
- Keycloak is self-hosted with its own PostgreSQL database; `deploy` provides
  it for local development (Docker Compose) and the cluster (Helm).
- Services and the gateway use only standard OIDC (discovery, JWKS, standard
  claims) configured through environment variables (issuer URL, client IDs),
  not Keycloak-specific APIs, so the provider stays replaceable.

## Consequences

- No password storage, reset, lockout or signing-key rotation code in
  CompanyCentral; that security-critical work is delegated to Keycloak.
- The OAuth2/OIDC IAM criterion of the RSO course is met.
- Keycloak is one more stateful component to deploy, upgrade, back up and
  monitor, with its own database. Realm and client configuration must be kept
  as code in `deploy` so environments are reproducible.
- User data is split: profile and credentials live in Keycloak, companies
  and memberships in `service-identity`. Invitations must link a Keycloak
  account to the invited user on first sign-in.
- Authentication in `service-identity` and `gateway` can now be implemented
  according to this record.
