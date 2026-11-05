# Deployment

## Target platform

The service runs on AWS using Lambda, SQS, and API Gateway. Face matching is
provided by Amazon Rekognition. Member data and photos remain in Supabase. No
VPC, subnets, NAT gateways, or databases are used on the AWS side.

## Components

- **API Gateway (HTTP API):** public HTTPS entry point with throttling and a JWT
  authorizer on the search route.
- **API Lambda (Python 3.12):** FastAPI application via Mangum.
- **Worker Lambda (Python 3.12):** SQS-triggered enrolment worker.
- **SQS standard queue and dead-letter queue:** the visibility timeout exceeds
  the worker timeout; jobs move to the dead-letter queue after a few failed
  receives.
- **Amazon Rekognition:** one face collection per environment.
- **SSM Parameter Store (SecureString):** Supabase URL and service key, webhook
  secret, and tunable settings such as the similarity threshold.
- **CloudWatch:** logs, metrics, and alarms.
- **EventBridge schedule (optional, later):** periodically re-queues members whose
  enrolment is pending or failed.

## IAM

| Role   | Permissions                                                                 |
| ------ | --------------------------------------------------------------------------- |
| API    | `rekognition:SearchFacesByImage`, `rekognition:DetectFaces`, `sqs:SendMessage` on the queue, read of its SSM parameters, log writes. |
| Worker | `rekognition:IndexFaces`, `rekognition:DeleteFaces`, `rekognition:DetectFaces`, SQS receive and delete on the queue, read of its SSM parameters, log writes. |

Bedrock permissions are added only when a feature requires them.

## Supabase-side setup

- Columns on the member record for enrolment state: `face_status`, `face_id`,
  and an optional error note.
- A database webhook on insert, update, and delete of members, targeting the
  webhook route and sending the shared secret header.
- A private storage bucket for member photos.
- Row-level security so staff can read member details after search.

## Environments

| Environment | Purpose                       |
| ----------- | ----------------------------- |
| `dev`       | Integration and early testing |
| `staging`   | Production-like verification  |
| `prod`      | Live traffic                  |

Each environment has its own Rekognition collection, queue, parameters, and
Supabase project or schema. Code is identical; only configuration differs.

## Infrastructure as code

All infrastructure is defined in `infra/` using AWS CDK in Python and deployed
through the pipeline. Manual console changes are not part of the process.

## Delivery pipeline

1. Pull request: lint, type check, unit tests.
2. Merge to main: package both functions and run tests.
3. Deploy to `dev` automatically, then `staging`.
4. Promotion to `prod` after approval, using Lambda aliases with gradual traffic
   shifting and automatic rollback on alarms.

## Operational notes

- Confirm Rekognition availability and quotas in the chosen region.
- Cap worker concurrency so a bulk import of members does not exceed Rekognition
  request limits; the queue absorbs the backlog.
- Alarm on any message in the dead-letter queue and on queue age.
- Clients downscale photos before upload because Lambda accepts at most 6 MB
  per request.
- Cold starts add latency to the first request after idle periods.

## Future hardening

If network isolation is required, the functions can be attached to a VPC with
interface endpoints. This is a contained change in `infra/`.
