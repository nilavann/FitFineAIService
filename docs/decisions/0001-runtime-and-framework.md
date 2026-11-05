# 0001. Python and FastAPI

**Status:** Accepted

## Context
The service is an API-only backend whose main work is calling a model provider
and validating structured output. Strong typing at the HTTP boundary and mature
AWS SDK support are the main requirements.

## Decision
Use Python 3.12 with FastAPI and Pydantic, adapted to Lambda with Mangum. Use `boto3` for AWS access.

## Consequences
- Request and response contracts are enforced by typed models, and OpenAPI is
  generated automatically.
- Async support suits long-running model calls.
- Python is the language with the broadest AWS and AI library coverage.
