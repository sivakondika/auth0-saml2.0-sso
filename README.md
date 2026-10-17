# Auth0 SAML 2.0 SSO Demo

A small Node.js service provider (SP) demo using Auth0 as its SAML identity provider (IdP). It supports SP-initiated and IdP-initiated sign-in.

## Requirements

- Node.js 18 or later
- An Auth0 tenant, a SAML application, and a test user

## Run locally

1. From `back-end`, install dependencies with `npm install`.
2. Create `back-end/.env` with the values below:

   ```env
   PORT=3000
   BASE_URL=http://localhost:3000
   SAML_ISSUER=http://localhost:3000/saml/metadata
   SAML_ENTRY_POINT=https://YOUR_TENANT.auth0.com/samlp/YOUR_CLIENT_ID
   SAML_IDP_CERT_PATH=./config/auth0-idp-cert.pem
   SESSION_SECRET=use-a-long-random-value
   ```

   `SAML_ENTRY_POINT` is Auth0's **Identity Provider Login URL**. Save Auth0's signing certificate as `back-end/config/auth0-idp-cert.pem`. Do not commit `.env` or credentials.

3. In Auth0, enable **SAML2 Web App** for your application. Set the callback URL to `http://localhost:3000/saml/acs` and use this addon Settings JSON:

   ```json
   {
     "audience": "http://localhost:3000/saml/metadata",
     "recipient": "http://localhost:3000/saml/acs",
     "destination": "http://localhost:3000/saml/acs",
     "mappings": {
       "email": "email",
       "name": "name",
       "given_name": "given_name",
       "family_name": "family_name"
     }
   }
   ```

   These values must match the SP configuration: `audience` is the SP Entity ID (`SAML_ISSUER`); `recipient` and `destination` are the Assertion Consumer Service (ACS) URL. JSON does not support comments, so keep explanations outside the Settings object.

4. Start the backend with `npm start` from `back-end`, then open `http://localhost:3000`.

## Troubleshooting: “SAML assertion has no AudienceRestriction”

The assertion from Auth0 must contain an `<AudienceRestriction>` whose audience is the SP Entity ID. The Node.js SP checks for this; setting `audience` only in the SP does not add the missing condition to Auth0's response.

- Set Auth0 SAML addon `audience` to exactly the same URL as `SAML_ISSUER` (by default, `http://localhost:3000/saml/metadata`).
- Save the addon settings and initiate a fresh login to get a new SAML response.
- If it still fails, inspect/decode the new response and confirm that the assertion includes `<Conditions><AudienceRestriction><Audience>http://localhost:3000/saml/metadata</Audience></AudienceRestriction></Conditions>`.
- Confirm `SAML_ISSUER` and the Auth0 `audience` do not differ by scheme, host, path, or trailing slash.

The Auth0 addon Settings value must be valid JSON: do not paste comments into it.

## Sign-in and validation

- **SP-initiated:** visit `/auth/login`; Auth0 returns its SAML response to `POST /saml/acs`.
- **IdP-initiated:** visit `/auth/idp-initiated`; Auth0 sends an unsolicited response to the same ACS endpoint. This flow has weaker request-correlation/CSRF protections; prefer SP-initiated sign-in where possible.
- **Metadata:** `/saml/metadata` publishes the SP Entity ID and ACS endpoint.
- **Session identity:** `/api/me` returns the signed-in user's demo profile.
- **Logout:** `/auth/logout` clears the local session; SAML Single Logout is not implemented.

The SAML library validates signatures, audience, recipient, and time conditions. This demo sets `validateInResponseTo: false` to accept IdP-initiated responses, so it does not enforce request correlation. The app also rejects recently reused assertion IDs using an in-memory replay cache. Sessions and replay records are lost on restart; multi-instance deployments need shared storage.
