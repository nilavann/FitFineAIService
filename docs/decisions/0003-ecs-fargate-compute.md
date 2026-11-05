# 0003. ECS on Fargate for compute

**Status:** Accepted

## Context
Model calls can run for many seconds. API Gateway's integration timeout and
Lambda's execution model make long or streamed responses awkward, and a
standard FastAPI container is the simplest unit to run and test locally.

## Decision
Run the service as a container on ECS Fargate behind an Application Load
Balancer.

## Alternatives considered
- **Lambda with API Gateway:** lower idle cost, but timeout limits and cold
  starts complicate long-running and streaming responses.
- **EKS:** more capability than this service needs, with higher operational cost.

## Consequences
- Local and deployed environments run the same image.
- Streaming responses are straightforward.
- A baseline cost exists for always-running tasks.
