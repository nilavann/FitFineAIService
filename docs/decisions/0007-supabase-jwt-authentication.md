# 0007. Supabase access tokens for search authentication

**Status:** Accepted

## Context
Staff already sign in through Supabase. The search endpoint must require a
signed-in user and be attributable to a person, because it handles biometric
lookups. The Supabase project currently signs tokens with the legacy shared
secret (HS256) rather than asymmetric keys.

## Decision
Callers send their Supabase access token. The API Lambda verifies the signature
with the project's JWT secret (kept in SSM), and checks expiry and audience.
Any authenticated user is accepted; no role is checked. The user's identifier is
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
- **Risk:** if members sign in to the same Supabase project, any member can call
  search. A staff role check, or a separate staff table lookup, closes this and
  can be added without changing the API.
