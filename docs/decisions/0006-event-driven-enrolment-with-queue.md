# 0006. Event-driven enrolment through a queue

**Status:** Accepted

## Context
Member photos live in Supabase. Embeddings must stay current when a photo is
added, replaced, or removed. Supabase database webhooks have a short timeout
and are not a reliable delivery mechanism on their own, while enrolment
involves a photo download, a Rekognition call, and a write-back.

## Decision
A Supabase webhook calls the API Lambda, which verifies the request and places
a job on an SQS queue. A separate Worker Lambda processes jobs. Jobs carry
references (member identifier, storage path, update time), not image data. The
worker re-reads current state, so stale or repeated jobs are harmless. Failed
jobs retry and then move to a dead-letter queue.

## Alternatives considered
- **Synchronous processing inside the webhook call:** fewer parts, but a timeout
  or error loses the event and leaves a member silently unsearchable.
- **Sending the photo or a signed URL in the event:** photos are large for a
  queue, and signed URLs can expire before a retry runs.

## Consequences
- Reliable retries and visible failures at the cost of one queue and one extra
  function.
- Enrolment is eventually consistent; `face_status` tells staff when it is ready.
- The write-back to Supabase would re-trigger the webhook, so the intake route
  only enqueues when the photo path actually changed.
