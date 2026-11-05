# Deployment

## Target platform

The service runs on AWS as a single Lambda function exposed through an Amazon
API Gateway HTTP API. Model inference is provided by Amazon Bedrock; no model
hosting is managed by this project. No VPC, subnets, or NAT gateways are used.

## Components

- **API Gateway (HTTP API):** public HTTPS entry point; routes all requests to
  the function and applies throttling.
- **Lambda function (Python 3.12):** runs the FastAPI application via Mangum.
- **IAM execution role:** grants only `bedrock:InvokeModel` on the approved
  models, plus permission to write logs.
- **CloudWatch Logs and metrics:** structured logs, error and latency alarms.

## Service calls

The function calls only the AWS services it needs, directly over their public
endpoints: Bedrock for inference and CloudWatch for logging. There is no
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

- Bedrock model access must be enabled per account and region before first use.
- Bedrock service quotas should be reviewed ahead of production load.
- The HTTP API integration timeout (30 seconds) bounds model call duration;
  set the Lambda timeout just under it.
- Cold starts add latency to the first request after idle periods; provisioned
  concurrency can be added later if this matters.

## Future hardening

If network isolation becomes a requirement, the function can be attached to a
VPC with a Bedrock interface endpoint. This is a contained change in `infra/`.
