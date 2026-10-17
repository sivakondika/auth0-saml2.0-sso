# Auth0 SAML SSO — Design Decisions

## 1. Overview

This project uses Auth0 as the SAML identity provider (IdP) and a Node.js application as the service provider (SP). It supports both SP-initiated and IdP-initiated sign-in.

After a successful SAML response, the application creates a local session and displays the authenticated user's identity.

## 2. Key Decisions and Assumptions

- **IdP and SP:** Auth0 is the IdP; the Node.js application is the SP.
- **SAML library:** Use `passport-saml` rather than implementing XML signature validation manually.
- **Endpoints:** The application provides login, metadata, Assertion Consumer Service (ACS), and authenticated-user endpoints. See the README for their URLs.
- **Attributes:** Attribute mappings and the NameID strategy are configured between Auth0 and the application.
- **Sessions:** Use `express-session` for this demo; persistent user storage is not required.
- **Identity:** Treat NameID as the external identity identifier. Email and display name are profile attributes that may change.
- **Test environment:** Use a development Auth0 tenant and synthetic test users.
- **Out of scope:** Persistent user storage, just-in-time (JIT) provisioning, role-based access control, multiple IdPs, and production certificate lifecycle management.
- **Logout:** The demo clears the local session only. SAML Single Logout (SLO) is not implemented.

## 3. Authentication Flows

### SP-initiated SSO

1. The user selects the SP sign-in option in the application.
2. The backend creates a SAML authentication request and redirects the browser to Auth0.
3. Auth0 authenticates the user and posts a SAML response to the ACS endpoint.
4. The backend validates the response and creates a local session.
5. The browser returns to the application and displays the authenticated identity.

### IdP-initiated SSO

1. The user starts at Auth0's configured Identity Provider Login URL.
2. Auth0 authenticates the user and posts an unsolicited SAML response to the ACS endpoint.
3. The backend validates the response and creates a local session without a preceding SP authentication request.

IdP-initiated SSO has weaker request-correlation and CSRF protections than SP-initiated SSO. Prefer SP-initiated sign-in when the application controls the entry point.

## 4. SAML Validation and Security

The application must not create a session unless SAML processing succeeds. The configured SAML library and application checks are responsible for validating the response, including its signature, audience, recipient, and time conditions.

- `validateInResponseTo` is currently set to `false` so the application can accept unsolicited IdP-initiated responses. Therefore, request correlation is **not enforced**, including for SP-initiated responses.
- The application tracks recently accepted assertion identifiers in an in-memory replay cache and rejects duplicates.
- Protect authenticated-user endpoints so unauthenticated requests cannot retrieve identity information.
- Keep secrets in environment variables; do not commit `.env` files or credentials.
- Use appropriate session-cookie attributes for the deployment environment.
- Do not log raw SAML assertions, credentials, tokens, or session secrets.
- Before production, review CSRF, session fixation, XSS, CORS, open redirects, dependency vulnerabilities, and HTTPS configuration.

Recommended negative tests include invalid signatures, expired assertions, incorrect audiences or destinations, and replay attempts. Request-correlation tests apply only if `InResponseTo` validation is enabled. Document a check as complete only after it has actually been implemented and tested.

## 5. Session and Identity Strategy

The demo stores session data in the default in-memory session store. Sessions are lost when the server restarts and are not shared between application instances.

For production, use shared session storage, explicit expiration and invalidation policies, and secure cookie settings.

The recommended external identity key is:

> SAML issuer + NameID

Email and display name are profile attributes, not stable identifiers. A production system should keep the external identity separate from the application's internal user ID—for example, with `User` and `ExternalIdentity` records and a unique constraint on `(issuer, nameId)`.

JIT provisioning, account linking, profile synchronization, and deprovisioning need explicit business rules. Do not automatically link accounts based only on matching email addresses.

## 6. Logout

The application session and the Auth0 session are separate. Ending one does not necessarily end the other.

- **Implemented:** `/auth/logout` invalidates the local application session.
- **Not implemented:** SAML SLO. A future SP-initiated implementation would send a LogoutRequest and validate the LogoutResponse, if supported by the IdP. An IdP-initiated implementation would validate the LogoutRequest, invalidate the relevant local session, and return a LogoutResponse.

Describe SLO as a future improvement, not a completed feature, until it is implemented and tested.

## 7. Configuration and Open Questions

Configuration choices are documented in the README and environment settings:

- Auth0 entry point used for IdP-initiated SSO.
- SP Entity ID, ACS URL, metadata URL, and SAML binding.
- Required SAML attributes and NameID strategy.
- Session duration, cookie settings, and behavior after a server restart.
- Tests performed and known limitations.
- Whether a local walkthrough is sufficient or a public deployment is required.

## 8. AI Usage

AI tools were used for scaffolding, research, and documentation. Review and test generated code, especially SAML validation, replay protection, and session handling. Disclose AI assistance accurately and explain how the implementation was verified.

## 9. Future Improvements

Prioritized follow-up work:

1. Add integration and negative security tests for both sign-in flows.
2. Implement and test SAML SLO if supported by the IdP.
3. Move sessions and replay-protection state to shared storage such as Redis.
4. Add persistent users and controlled JIT provisioning.
5. Add structured authentication audit logs, monitoring, and alerting.
6. Add certificate rotation and production HTTPS/security hardening.
7. Add multiple-IdP support only if future requirements justify it.

## Summary

The demo supports SP-initiated and IdP-initiated SSO. It validates SAML responses before creating local sessions and documents its assumptions and limitations. Production use would require additional security tests, shared session and replay storage, user lifecycle management, monitoring, certificate rotation, and—if supported—SAML SLO.