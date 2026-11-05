# 0007. Supabase access tokens, verified with public signing keys

**Status:** Accepted

## Context
Staff already sign in through Supabase, and only staff hold accounts in the
project. The search and enrol endpoints must require a signed-in user and be
attributable to a person, because search is a biometric lookup. The Supabase
project uses JWT signing keys (asymmetric), which publish their public keys at
`/auth/v1/.well-known/jwks.json`.

## Decision
Callers send their Supabase access token. The API Lambda verifies it using the
project's public keys:

- Keys are fetched from the JWKS URL and cached; they are refetched when a token
  names an unknown key identifier.
- Only the asymmetric algorithm the project uses is accepted. Symmetric
  algorithms are rejected, which prevents algorithm-confusion attacks.
- Issuer, audience (`authenticated`), and expiry are checked.
- Any valid token is accepted; no role is checked, because every account in the
  project belongs to staff. The user's identifier is written to the request log.

The webhook route is separate and uses a shared secret header.

## Alternatives considered
- **API Gateway's built-in JWT authorizer:** would reject bad requests before they
  reach the function, but it depends on an OpenID discovery document that
  Supabase's documentation does not describe. It can be adopted later if it is
  confirmed to work.
- **Custom authorizer function:** adds a function and cold start for little gain
  over verifying in the API Lambda.
- **Legacy shared JWT secret:** Supabase recommends moving off it, and it would
  need a secret stored in SSM.
- **Shared API key in the app:** cannot identify individuals and is hard to rotate.

## Consequences
- No signing secret to store or rotate on our side; only public keys are used.
- Unauthenticated requests reach the function before being rejected. Throttling
  limits the cost.
- Verification sits in one function, so adopting the API Gateway authorizer later
  is a contained change.
- **Assumption:** every account in the project is a staff account. If member or
  other non-staff sign-ins are ever added, a staff check (a role claim or a staff
  table lookup) must be added first. This can be done without changing the API.
