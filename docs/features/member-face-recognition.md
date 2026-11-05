# Member face recognition

## Purpose

Allow staff to find a member by photo. A photo is enrolled against a member
when the member is added; later, a staff member submits a photo and the service
returns the matching member's details.

## Flows

### Enrolment
1. A client submits a photo with the member identifier and member details.
2. The service checks that the photo contains exactly one usable face.
3. The face is indexed in the Rekognition collection, tagged with the member
   identifier.
4. The member details and the returned face identifier are stored in DynamoDB.
5. Re-enrolling a member replaces the previous face.

### Lookup
1. A client submits a photo.
2. The service searches the collection for the closest enrolled faces above a
   similarity threshold.
3. For each match, the service loads member details and returns them with the
   similarity score.

## API (v1)

| Method and path                  | Purpose                                      |
| -------------------------------- | -------------------------------------------- |
| `POST /v1/members/{id}/face`     | Enrol or replace a member's photo and details|
| `DELETE /v1/members/{id}/face`   | Remove the member's face from the collection |
| `POST /v1/members/search`        | Find members matching a submitted photo      |

### Enrolment request
Multipart form with `photo` (JPEG or PNG) and `details` (JSON: name, contact,
membership fields as agreed).

### Search response
```json
{
  "matches": [
    { "member_id": "m_1042", "similarity": 98.7, "member": { "name": "..." } }
  ]
}
```
An empty `matches` array means no member met the threshold.

## Data model

| Store         | Content                                                        |
| ------------- | -------------------------------------------------------------- |
| Rekognition   | Face vectors, with `member_id` as the external image identifier|
| DynamoDB      | `member_id` (key), member details, `face_id`, enrolment time   |

## Behaviour and limits

- Results are suggestions for staff to confirm, not automated decisions. The
  service returns up to three candidates ordered by similarity.
- The similarity threshold is configuration. It starts high and is tuned using
  real photos from the gym's entrance conditions.
- Photos with no face, several faces, or poor quality are rejected with a clear
  error code.
- Photo size: clients downscale before upload because Lambda accepts at most
  6 MB per request.
- Requests with a printed or on-screen photo can fool matching; liveness
  checking is not included and is a candidate for later work.

## Privacy and compliance

Face data is biometric and is treated as sensitive personal data.

- Collect explicit consent before enrolment and record that it was given.
- Provide deletion: removing a member removes their face from the collection
  and their stored record.
- Keep source photos out of logs; log identifiers only.
- Restrict both endpoints to authenticated staff, and record who searched.
- Confirm the legal requirements of the operating jurisdiction before launch.

## Open questions

- Is the member record owned by this service, or does another system hold it and
  this service stores only the identifier and face?
- Authentication method for staff (see architecture).
- Retention period for source photos, if they are kept at all.
- Expected member count, to confirm collection and cost estimates.
