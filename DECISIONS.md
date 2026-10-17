Auth0 SAML SSO --- Design Decisions
1. Overview
This project integrates Auth0 as the SAML Identity Provider (IdP)
and a Node.js application as the SAML Service Provider (SP).
Both required login flows are implemented: - SP-initiated SSO: Login
starts from the application and redirects the user to Auth0. -
IdP-initiated SSO: Login starts from the configured Auth0 SSO entry
point and returns the SAML response to the application's Assertion
Consumer Service (ACS).
After successful SAML processing, the application creates a local
session and displays the authenticated user's identity.
2. Key Decisions & Assumptions
- IdP/SP roles: Auth0 acts as the IdP; the Node.js application
  acts as the SP.
- SAML library: Use a maintained SAML library rather than
  implementing XML signature validation manually.
- Endpoints: The application defines its own metadata, login, ACS,
  and authenticated-user routes. Their actual URLs are documented in
  the README.
- SAML binding and attributes: Use the binding and attribute
  mapping configured in Auth0 and the application. Document the
  selected values in the README.
- Session: Use a lightweight session strategy suitable for the
  demo; no database is required.
- Identity attributes: Treat NameID as the external identity
  identifier and email/name as profile attributes.
- Test environment: Use a development Auth0 tenant and synthetic
  test users. Local execution and a walkthrough are sufficient unless
  a public deployment is explicitly requested.
- Scope: Persistent user storage, JIT provisioning, RBAC,
  multi-IdP support, and production certificate lifecycle management
  are out of scope.
- Logout: SAML Single Logout is an optional stretch goal. The
  intended design is documented below; implementation status should be
  confirmed in the README.
3. Authentication Flows
SP-Initiated SSO
1. The user selects Login with SSO in the application.
2. The backend initiates a SAML authentication request and redirects
   the browser to Auth0.
3. Auth0 authenticates the user and posts the SAML response to the ACS
   endpoint.
4. The backend validates and processes the response, then creates the
   application session.
5. The user returns to the application and sees their authenticated
   identity.
IdP-Initiated SSO
1. The user starts from the configured Auth0 SSO entry point.
2. Auth0 authenticates the user and sends a SAML response to the ACS
   endpoint.
3. The backend processes the unsolicited response without requiring a
   preceding SP-generated authentication request.
4. The application creates a local session and displays the
   authenticated identity.
The two entry points are documented so reviewers can test each flow
independently.
4. SAML Validation & Security
A session must only be created after successful SAML response
validation. The implementation should use the selected SAML library to
validate the trusted issuer and signature, audience,
recipient/destination, assertion validity conditions, and subject
confirmation.
- For SP-initiated SSO, validate InResponseTo against the
  outstanding authentication request.
- Reject assertion IDs that have already been processed.
- For IdP-initiated SSO, handle the absence of InResponseTo
  deliberately while retaining all applicable signature, issuer,
  audience, destination, and time validations.
- Protect the authenticated-user endpoint so unauthenticated requests
  cannot retrieve identity information.
- Keep secrets in environment variables and commit only
  .env.example.
- Use appropriate session-cookie attributes for the environment.
- Do not log raw SAML assertions, credentials, tokens, or session
  secrets.
- Consider CSRF, session fixation, XSS, CORS, open redirects,
  dependency vulnerabilities, and HTTPS before production deployment.
Negative tests should cover invalid signatures, expired assertions,
incorrect audience/destination, invalid request correlation, and replay
attempts. Only describe checks as complete in the README if they have
actually been implemented and tested.
5. Session & Identity Strategy
The demo does not require persistent user storage. The application
creates a temporary session after successful SAML authentication. With
in-memory storage, sessions are lost on server restart and are not
shared across multiple application instances.
For production, I would use shared session storage, defined expiration
and invalidation policies, and secure cookie settings.
The recommended external identity key is:
SAML Issuer + NameID
Email and display name are profile attributes because they can change. A
production system should keep the external identity separate from the
application's internal user ID, for example through User and
ExternalIdentity records with a unique constraint on
(issuer, nameId).
JIT provisioning, account linking, profile synchronization, and
deprovisioning require explicit business rules. Accounts should not be
linked automatically based only on matching email addresses.
6. Logout Design
The application session and Auth0 session are separate; ending one does
not guarantee that the other has ended.
- SP-initiated logout: Invalidate the local session first. If
  Auth0 and the SAML configuration support Single Logout, send a SAML
  LogoutRequest and validate the corresponding LogoutResponse.
- IdP-initiated logout: Validate the incoming SAML
  LogoutRequest, identify and invalidate the relevant local session,
  and return a valid LogoutResponse.
Full SLO depends on Auth0 support and configuration. If it is not
implemented, treat it as a documented future improvement rather than a
completed feature.
7. Gaps & Clarifications
The assignment leaves some implementation choices open. This solution
makes them explicit in the README and configuration:
- The Auth0 entry point used to demonstrate IdP-initiated SSO.
- The configured SP Entity ID, ACS URL, metadata URL, and SAML
  binding.
- Required SAML attributes and the NameID strategy.
- Session duration, cookie settings, and behavior after a server
  restart.
- The test cases executed and any known limitations.
- Whether reviewers will use a local demo and walkthrough or require a
  public deployment.
8. AI Usage
AI tools used for scaffolding, research, and documentation. All
generated code must be reviewed and tested, especially SAML validation,
replay protection, and session handling. The submission should
accurately disclose AI assistance and explain how the implementation was verified.
9. Improvements with More Time
I would prioritize improvements in this order:
1. Add automated integration and negative security tests for both login
   flows.
2. Implement and test SAML Single Logout if supported by the IdP
   configuration.
3. Move sessions and replay-protection state to a shared store such as
   Redis.
4. Add persistent users and controlled JIT provisioning.
5. Introduce structured authentication audit logs, monitoring, and
   alerting.
6. Support certificate rotation and production HTTPS/security
   hardening.
7. Add customer-specific or multiple-IdP configuration only if future
   business requirements justify it.
Summary
The demo implements both SP-initiated and IdP-initiated SSO. The design
prioritizes clear authentication flows, SAML validation before session
creation, and documented assumptions. Production readiness would require
additional automated security testing, shared session and replay
storage, user lifecycle management, observability, certificate rotation,
and---where supported---complete SAML Single Logout.