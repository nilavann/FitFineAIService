# 0003. Lambda behind API Gateway for compute

**Status:** Accepted

## Context
The service is a small, API-only backend. Early traffic is expected to be low
and variable, so minimising fixed cost and operational surface matters more
than long-running or streamed responses.

## Decision
Run the service as a Lambda function behind an API Gateway HTTP API. The
function is not attached to a VPC and calls AWS services directly.

## Alternatives considered
- **ECS on Fargate with an ALB:** supports long and streamed responses, but
  carries a continuous baseline cost and more infrastructure to manage.
- **Lambda inside a VPC:** adds subnets, and a NAT gateway or endpoints, with
  recurring cost and no current requirement for them.

## Consequences
- Pay-per-request cost with no idle charge, and no networking to operate.
- Model calls must finish within the 30-second HTTP API limit; streaming and
  long-running work are out of scope for now.
- Cold starts affect first-request latency.
- Moving to containers or a VPC later is possible without changing the API.
