# Member face recognition

## Purpose

Allow staff to find a member from a photo. Enrolment is driven by changes in
Supabase: when a member's photo is added, replaced, or removed, the service
updates that member's face embedding without any manual step.

## Enrolment flow

```
Supabase member row changes
  → webhook → API Lambda: verify secret, decide if the photo changed, enqueue job
  → SQS → Worker Lambda:
       1. read the member's current row from Supabase
       2. compare photo path with the job; skip if the row has moved on
       3. create a short-lived signed URL and download the photo
       4. check the photo has exactly one usable face
       5. delete the member's previous face, then index the new one
       6. write face_status and face_id back to Supabase
```

### Rules

- **Jobs carry references only:** `member_id`, the photo storage path, and the
  row's `updated_at`. Image bytes never travel through the webhook or queue, and
  signed URLs are created at processing time because they expire.
- **Write-back must not loop.** The status update changes the member row and
  would fire the webhook again. The webhook route enqueues a job only when the
  photo path differs between the old and new record (or on insert or delete).
- **Stale events are skipped.** The worker acts on the row's current state.
- **Removal:** when the photo is cleared or the member is deleted, the worker
  deletes the stored face from Rekognition.
- **Replacement:** the previous face is deleted using the stored `face_id`
  before the new one is indexed, so a member never has two faces.
- **Idempotent:** repeating a job leaves the same result.

### Enrolment status

| `face_status` | Meaning                                              |
| ------------- | ---------------------------------------------------- |
| `pending`     | Photo change received, not yet processed             |
| `ready`       | Face indexed and searchable                          |
| `no_face`     | Photo had no usable face; a new photo is needed      |
| `multiple_faces` | Photo had several faces; a new photo is needed    |
| `failed`      | Processing failed after retries; see the dead-letter queue |

## Search flow

```
Staff app → API Gateway → API Lambda (token verified) → Rekognition → matches
```

1. The staff app sends a photo with the member's Supabase access token.
2. The function verifies the token's signature, expiry, and audience. Any signed-in
   user is accepted; there is no role check.
3. The photo is checked for exactly one face, then searched against the
   collection.
4. The function returns up to three candidates above the similarity threshold.

The service returns identifiers and scores only. The app loads member details
from Supabase under its own row-level security.

## API (v1)

| Method and path                      | Caller   | Purpose                                      |
| ------------------------------------ | -------- | -------------------------------------------- |
| `POST /v1/webhooks/member-photo`     | Supabase | Receive member change events                 |
| `POST /v1/members/search`            | Staff    | Find members matching a submitted photo      |
| `GET /health`                        | Monitor  | Liveness check                               |

### Webhook event (fields used)
```json
{
  "type": "UPDATE",
  "record":     { "id": "m_1042", "photo_path": "members/m_1042/new.jpg", "updated_at": "..." },
  "old_record": { "id": "m_1042", "photo_path": "members/m_1042/old.jpg" }
}
```

### Search request and response
Request: multipart form with `photo` (JPEG or PNG).
```json
{ "matches": [ { "member_id": "m_1042", "similarity": 98.7 } ] }
```
An empty `matches` array means no member met the threshold.

## Initial backfill

Existing members are enrolled by `scripts/backfill.py`. It selects members that
have a photo and no `ready` status, sets them to `pending`, and queues one job
each, the same job shape the webhook produces. Worker concurrency is capped so
Rekognition limits are respected. The script is safe to re-run. It should run
after the webhook is live so that photo changes made during the backfill are
not lost.

## Data

| Store       | Content                                                             |
| ----------- | ------------------------------------------------------------------- |
| Supabase    | Member details, photo, `face_status`, `face_id`                     |
| Rekognition | Face embeddings, with `member_id` as the external image identifier  |

## Behaviour and limits

- Results are suggestions for staff to confirm, not automated decisions.
- The similarity threshold is configuration. It starts high and is tuned with
  real photos taken in the gym's entrance conditions.
- Photos with no face, several faces, or poor quality are rejected with a clear
  error (search) or a status (enrolment).
- Printed or on-screen photos can fool matching; liveness checking is not
  included and is a candidate for later work.

## Privacy and compliance

Face data is biometric and is treated as sensitive personal data.

- Collect explicit consent before enrolment and record it on the member.
- Deletion of a member or photo removes the face from the collection.
- Keep photos and embeddings out of logs; log identifiers only.
- Restrict search to signed-in users and record who searched.
- Confirm the legal requirements of the operating jurisdiction before launch.

## Open questions

- Whether members share the Supabase project's sign-in (see ADR 0007).
- Retention and consent-record design in Supabase.
- Expected member count, to confirm cost estimates.
- Number of existing members to backfill, for time and cost estimates.
