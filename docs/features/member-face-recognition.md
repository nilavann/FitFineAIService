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
  photo path or the consent timestamp differs between the old and new record (or
  on insert or delete).
- **Consent is required.** The worker enrols a face only if the member's
  `face_consent_at` is set. Without it the member gets `face_status = no_consent`
  and no face is stored. If consent is later withdrawn (the timestamp is cleared),
  the change queues a job and the worker removes the existing face.
- **Stale events are skipped.** The worker acts on the row's current state. Jobs
  from the enrol route carry no photo path, so the worker uses whatever photo the
  row holds at that moment.
- **Removal:** when the photo is cleared, consent is withdrawn, or the member is
  deleted, the worker deletes the stored face from Rekognition (see Deletion).
- **Replacement:** the previous face is deleted using the stored `face_id`
  before the new one is indexed, so a member never has two faces.
- **Idempotent:** repeating a job leaves the same result.

### Enrolment status

| `face_status` | Meaning                                              |
| ------------- | ---------------------------------------------------- |
| `pending`     | Photo change received, not yet processed             |
| `ready`       | Face indexed and searchable                          |
| `no_consent`  | No consent recorded; nothing is stored               |
| `no_face`     | Photo had no usable face; a new photo is needed      |
| `multiple_faces` | Photo had several faces; a new photo is needed    |
| `failed`      | Processing failed after retries; see the dead-letter queue |

### Deletion

Face data is removed immediately when a member is deleted, so no biometric data
outlives the member.

```
Supabase member row deleted
  → database trigger/webhook (DELETE) → API Lambda: verify secret, enqueue delete job
  → SQS → Worker Lambda: delete the face from Rekognition
```

- The row is gone by the time the worker runs, so the **delete job carries the
  `face_id`** taken from the deleted row (`old_record`). A member who was never
  enrolled has no `face_id`, and the job is skipped.
- Deleting a face that no longer exists is not an error, so retries are safe.
- Nothing can be written back to Supabase for a deleted row; the outcome is
  logged by member identifier.
- A delete that keeps failing lands in the dead-letter queue, which has an alarm,
  because failing to remove biometric data needs attention.
- Postgres cannot write to SQS directly without signing AWS requests. The trigger
  therefore calls the API route over HTTP, and the API Lambda queues the job.

## Search flow

```
Staff app → API Gateway → API Lambda (token verified) → Rekognition → matches
```

1. The staff app sends a photo with the member's Supabase access token.
2. The function verifies the token's signature (against Supabase's public keys), issuer, audience, and expiry. Only staff have
   accounts in the project, so any valid token is accepted; there is no role check.
3. The photo is checked for exactly one face, then searched against the
   collection.
4. The function returns up to three candidates above the similarity threshold.

The service returns identifiers and scores only. The app loads member details
from Supabase under its own row-level security.

## API (v1)

| Method and path                      | Caller   | Purpose                                      |
| ------------------------------------ | -------- | -------------------------------------------- |
| `POST /v1/webhooks/member-photo`     | Supabase | Receive member change events                 |
| `POST /v1/members/{id}/enrol`        | Staff app| Queue enrolment for one existing member      |
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

### Enrol request and response
`POST /v1/members/{id}/enrol` takes a Supabase access token and no body. It
queues a job containing only the `member_id` and replies `202`:
```json
{ "member_id": "m_1042", "status": "queued" }
```
The worker then reads the member's current photo from Supabase and enrols it
through the same steps as any other job. The route does not check the photo
itself; problems appear later as the member's `face_status`. Calling it again
is safe, because enrolment replaces any previous face.

Intended use: enrolling members who already exist, or re-running enrolment for
a member whose status is `failed`, `no_face`, or `multiple_faces`. The app calls
it once per member; throttling on API Gateway and the queue absorb a large run.

### Search request and response
Request: multipart form with `photo` (JPEG or PNG).
```json
{ "matches": [ { "member_id": "m_1042", "similarity": 98.7 } ] }
```
An empty `matches` array means no member met the threshold.

## Data

| Store       | Content                                                             |
| ----------- | ------------------------------------------------------------------- |
| Supabase    | Member details, photo, `face_status`, `face_id`                     |
| Rekognition | Face embeddings, with `member_id` as the external image identifier  |

## Scale

About 44 members. Rekognition storage and request volume at this size are
negligible, so cost is not a design constraint for this feature.

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
- Deleting a member, clearing the photo, or withdrawing consent removes the face
  from the collection immediately.
- Keep photos and embeddings out of logs; log identifiers only.
- Restrict search to signed-in users and record who searched.
- Confirm the legal requirements of the operating jurisdiction before launch.

## Open questions

- What counts as a member leaving: only deletion of the row, or also a status
  change such as `inactive` or `cancelled`? A status change needs its own rule in
  the webhook filter.
- The consent column name and how staff record consent in the app.


