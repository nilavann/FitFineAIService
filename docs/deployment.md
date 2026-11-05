# Deployment

## Target platform

The service runs on AWS as a single Lambda function exposed through an Amazon
API Gateway HTTP API. Face matching is provided by Amazon Rekognition and
member records are held in DynamoDB; no model hosting is managed by this project. No VPC, subnets, or NAT gateways are used.

## Components

- **API Gateway (HTTP API):** public HTTPS entry point; routes all requests to
  the function and applies throttling.
- **Lambda function (Python 3.12):** runs the FastAPI application via Mangum.
- **Amazon Rekognition:** one face collection per environment.
- **DynamoDB:** member table with on-demand capacity.
- **IAM execution role:** least privilege: the Rekognition collection actions
  (`IndexFaces`, `SearchFacesByImage`, `DeleteFaces`), read/write on the member
  table, and log writing. Bedrock permissions are added when a feature needs them.
- **CloudWatch Logs and metrics:** structured logs, error and latency alarms.

## Service calls

The function calls only the AWS services it needs, directly over their public
endpoints: Rekognition, DynamoDB, and CloudWatch. There is no
private networking layer.

## Environments

| Environment | Purpose                         |
| ----------- | ------------------------------- |
| `dev`       | Integration and early testing   |
| `staging`   | Production-like verification    |
| `prod`      | Live traffic                    |

Each environment is deployed from the same code; only configuration differs.

## Infrastructure as code

All infrastructure is defined in `infra/` using AWS CDK in Python and deployed
through the pipeline. Manual console changes are not part of the process.

## Delivery pipeline

1. Pull request: lint, type check, unit tests.
2. Merge to main: package the function and run tests.
3. Deploy to `dev` automatically, then `staging`.
4. Promotion to `prod` after approval, using a Lambda alias with gradual
   traffic shifting and automatic rollback on alarms.

## Operational notes

- Rekognition availability and quotas should be confirmed for the chosen region.
- Lambda accepts request payloads up to 6 MB, so clients should downscale photos
  (about 1600 px JPEG) before upload.
- The HTTP API integration timeout (30 seconds) bounds request duration;
  set the Lambda timeout just under it.
- Cold starts add latency to the first request after idle periods; provisioned
  concurrency can be added later if this matters.

## Future hardening

If network isolation becomes a requirement, the function can be attached to a
VPC with interface endpoints for the services it calls. This is a contained change in `infra/`.
