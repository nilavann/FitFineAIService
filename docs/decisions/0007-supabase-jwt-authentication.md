# 0007. Supabase access tokens for staff authentication

**Status:** Accepted

## Context
Staff already sign in through Supabase. The search endpoint must be limited to
staff and attributable to a person, because it handles biometric lookups.

## Decision
Staff call the search endpoint with their Supabase access token. API Gateway's
JWT authorizer verifies the signature, issuer, and audience, and the function
checks a staff role claim. The webhook route is separate and uses a shared
secret header instead.

## Alternatives considered
- **Shared API key in the app:** cannot identify individuals and is hard to
  rotate.
- **AWS IAM signing:** poorly suited to browser and mobile clients.

## Consequences
- No new identity system and per-person audit of searches.
- The built-in authorizer requires the Supabase project to publish asymmetric
  signing keys. A project using only a shared signing secret needs a small
  custom authorizer function instead.
- The claim identifying staff must be agreed and set in Supabase.
