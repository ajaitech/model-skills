---
name: aivibe-sso-cognito
description: The complete AWS Cognito single-sign-on architecture for the AiVibe platform — auth.aivibe.cloud (Hosted UI / OAuth2), cognito.aivibe.cloud/cognito (IDP proxy), api.aivibe.cloud (platform GraphQL), the shared user pool us-east-1_S2Cpx3svp, per-app client ids, all auth flows (PKCE browser SSO, password, email OTP, social, device grant, silent SSO, refresh), token handling, and the custom:tenant_id claim. Use for any login/auth/SSO/token/JWKS/tenant-claim task across aicippy, aivedha, vibekaro, or the platform.
---

# AiVibe SSO — AWS Cognito (grounded in the repos)

One Cognito user pool backs single sign-on for every AiVibe app. Login at
`auth.aivibe.cloud` once; all apps recognize the same `sub` and `custom:tenant_id`.

## Identifiers (real values)
- **User pool:** `us-east-1_S2Cpx3svp` (region **us-east-1**)
- **App clients:** AiCippy = `2gj7kdplhfg4aoenqfjghpotie`; AiVedha Guard = `513flu46nof8hk5p291esas7gj` (per-app clients, same pool)
- **Subdomain → id mapping** (`SSO_MAPPINGS`): `auth.aivibe.cloud`→pool;
  `aicippy-auth/aivedha-auth/vibekaro-auth/aipoha-auth/aikutty-auth.aivibe.cloud`
  all route to the same pool `us-east-1_S2Cpx3svp`.

## Endpoints
| URL | Role |
|---|---|
| `https://auth.aivibe.cloud/oauth2/authorize` | OAuth2 Authorization Code (+PKCE), social IdP, silent SSO |
| `https://auth.aivibe.cloud/oauth2/token` | code exchange + refresh + device poll |
| `https://auth.aivibe.cloud/oauth2/userInfo` | user info (Bearer) |
| `https://auth.aivibe.cloud/us-east-1_S2Cpx3svp/.well-known/jwks.json` | **JWKS** (verify tokens) |
| `https://auth.aivibe.cloud/us-east-1_S2Cpx3svp` | **issuer** (`iss`) |
| `https://cognito.aivibe.cloud/cognito` | **IDP proxy** — direct Cognito JSON-RPC |
| `https://api.aivibe.cloud/graphql` | platform API (Bearer id_token) |

IDP-proxy calls use: `Content-Type: application/x-amz-json-1.1` +
`X-Amz-Target: AWSCognitoIdentityProviderService.<Action>`.

## Auth flows (all implemented)
- **Browser SSO (Authorization Code + PKCE):** scopes `openid email profile`;
  CLI redirect `http://localhost:8975/callback`; extension redirect
  `https://<ext-id>.chromiumapp.org/`. PKCE S256 (`code_challenge_method=S256`).
- **Email + password:** `InitiateAuth` `AuthFlow=USER_PASSWORD_AUTH` (USERNAME/PASSWORD).
- **Email OTP:** `InitiateAuth` `AuthFlow=USER_AUTH` `PREFERRED_CHALLENGE=EMAIL_OTP`
  → `RespondToAuthChallenge` `ChallengeName=EMAIL_OTP` with `EMAIL_OTP_CODE`
  (CLI variant uses `CUSTOM_AUTH`/`CUSTOM_CHALLENGE` + `ANSWER`).
- **Social (Google / GitHub):** `/oauth2/authorize?...&identity_provider=Google|GitHub`
  via `chrome.identity.launchWebAuthFlow`; Guard also supports direct GitHub OAuth.
- **Device grant (headless/SSH):** `/oauth2/device_authorization` →
  poll `/oauth2/token` `grant_type=urn:ietf:params:oauth:grant-type:device_code`
  (handle `authorization_pending` / `slow_down` / `expired_token`).
- **Silent SSO:** `/oauth2/authorize` with `interactive:false` → code if a session exists.
- **Refresh:** `InitiateAuth` `AuthFlow=REFRESH_TOKEN_AUTH`; proactively refresh **5 min**
  before expiry.

## Tokens & claims
- **Storage:** CLI → OS keychain; extension → `chrome.storage.session` (fast) + `local`
  (survives restart, holds refresh_token); web → localStorage.
- **id_token** = identity (`sub`, `email`, **`custom:tenant_id`**, `cognito:username`);
  **access_token** = API authz (Bearer); **refresh_token** = renew.
- **Tenant claim:** `custom:tenant_id` (fallbacks seen: `custom:tenantId`, `custom:orgId`, `sub`).

## Verify / debug
- Validate a token: fetch the JWKS above, check signature + `iss` (issuer) + `aud`
  (the app client id) + `exp`.
- 401 from `api.aivibe.cloud` → expired/invalid id_token → refresh or re-login.
- Wrong-client errors → confirm you used that app's client id (subdomain mapping).
- A protected endpoint returning 500 on missing token is a backend bug — it should be 401.

## Backend enforcement (multi-tenant)
The edge validates the JWT and extracts `custom:tenant_id` — pass THAT downstream,
never a body tenant id. See aivibe-db-schema (tenant isolation) and
bedrock-multitenant-security.
