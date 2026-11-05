# 0007. Supabase access tokens for search authentication

**Status:** Accepted

## Context
Staff already sign in through Supabase, and only staff hold accounts in the project. The search endpoint must require a
signed-in user and be attributable to a person, because it handles biometric
lookups. The Supabase project currently signs tokens with the legacy shared
secret (HS256) rather than asymmetric keys.

## Decision
Callers send their Supabase access token. The API Lambda verifies the signature
with the project's JWT secret (kept in SSM), and checks expiry and audience.
Any valid token is accepted; no role is checked, because every account in the
project belongs to staff. The user's identifier is
written to the request log. The webhook route is separate and uses a shared
secret header.

## Alternatives considered
- **API Gateway's built-in JWT authorizer:** needs asymmetric signing keys, which
  the project does not use.
- **Separate custom authorizer function:** works with the legacy secret but adds
  a function and a cold start for little gain over verifying in the API Lambda.
- **Shared API key in the app:** cannot identify individuals and is hard to rotate.

## Consequences
- No new identity system and per-person audit of searches.
- The JWT secret is a sensitive value; it is stored in SSM and readable only by
  the API Lambda's role. Rotating it signs out all users.
- Verification sits in one function, so adopting Supabase's asymmetric signing
  keys later only changes that function.
- **Assumption:** every account in the project is a staff account. If member or
  other non-staff sign-ins are ever added, a staff check (a role claim or a staff
  table lookup) must be added first, since search is a biometric lookup. This can
  be done without changing the API.
