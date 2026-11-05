# 0009. Consent gating and immediate deletion of face data

**Status:** Accepted

## Context
Face embeddings are biometric data. The service should not hold them without the
member's consent or after the member has gone.

## Decision
- **Consent gating.** Enrolment proceeds only when `face_consent_at` is set on the
  member in Supabase. Otherwise the member's `face_status` is `no_consent` and
  nothing is stored. Clearing the timestamp removes the existing face.
- **Immediate deletion.** When a member row is deleted, a DELETE event carrying
  the old row, including `face_id`, goes to the API route, which queues a delete
  job. The worker removes the face. Clearing the photo is handled the same way.
- The existing database webhook (a trigger plus an HTTP call) delivers these
  events. Postgres does not write to SQS directly, so the trigger calls the API
  route and the API Lambda queues the job.

## Alternatives considered
- **Consent handled outside the system:** simpler, but the service could hold data
  for members who never agreed.
- **Delayed or on-request deletion:** leaves biometric data in place longer than
  necessary.
- **Postgres writing straight to SQS:** requires signing AWS requests from the
  database; the HTTP route is simpler and already authenticated by the shared
  secret.

## Consequences
- A new `face_consent_at` column and `no_consent` status in Supabase.
- The webhook filter watches both the photo path and the consent timestamp.
- Delete jobs carry `face_id` because the row no longer exists, and failures are
  alarmed through the dead-letter queue.
