# Deployment

## Target platform

The service runs on AWS. The application is built as a container image and
deployed to ECS on Fargate. Model inference is provided by Amazon Bedrock;
no model hosting is managed by this project.

## Topology

- **VPC** with public subnets for the load balancer and private subnets for
  tasks.
- **Application Load Balancer** terminating TLS and routing to the service.
- **ECS service (Fargate)** running the container across multiple availability
  zones, scaled on CPU and request count.
- **IAM task role** granting only `bedrock:InvokeModel` (and streaming
  equivalent) on the approved models, plus read access to its own secrets.
- **VPC endpoint for Bedrock** so model traffic stays on the AWS network.
- **ECR** repository holding versioned images.
- **CloudWatch** log groups, metrics, and alarms.
- **Secrets Manager / SSM Parameter Store** for configuration values.

## Environments

| Environment | Purpose                         |
| ----------- | ------------------------------- |
| `dev`       | Integration and early testing   |
| `staging`   | Production-like verification    |
| `prod`      | Live traffic                    |

Each environment is deployed from the same code and image; only configuration
differs. Separate AWS accounts per environment are preferred.

## Infrastructure as code

All infrastructure is defined in `infra/` using AWS CDK in Python and deployed
through the pipeline. Manual console changes are not part of the process.

## Delivery pipeline

1. Pull request: lint, type check, unit tests.
2. Merge to main: build the image, run tests, scan the image, push to ECR.
3. Deploy to `dev` automatically, then `staging`.
4. Promotion to `prod` after approval, using a rolling deployment with
   automatic rollback on failed health checks or alarms.

## Operational notes

- Bedrock model access must be enabled per account and region before first use.
- Bedrock service quotas should be reviewed and raised ahead of production load.
- Model calls can take several seconds, so load balancer idle timeouts and
  client timeouts must be set accordingly.
