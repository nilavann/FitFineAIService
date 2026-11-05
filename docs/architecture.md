# Architecture

## Purpose and scope

The service provides AI-backed functionality to FitFine clients through a
versioned HTTP API. The service owns request validation, prompt construction,
model invocation, and response validation. It does not own any user interface,
and it does not own long-term user data; callers supply the context needed for
each request.

## System context

```
 FitFine clients ──HTTPS──▶ ALB ──▶ ECS Fargate (FastAPI) ──▶ Amazon Bedrock
                                          │
                                          └──▶ CloudWatch (logs, metrics)
                                          └──▶ Secrets Manager / SSM (config)
```

## Layering

Code is organised so that each layer depends only on the one beneath it.

| Layer      | Location       | Responsibility                                          |
| ---------- | -------------- | ------------------------------------------------------- |
| API        | `app/api/`     | Routing, HTTP concerns, status codes. No business logic.|
| Schemas    | `app/schemas/` | Typed request and response models (Pydantic).           |
| Services   | `app/services/`| Feature logic; orchestrates prompts and model calls.    |
| AI         | `app/ai/`      | Bedrock client wrapper, prompt templates, output parsing.|
| Core       | `app/core/`    | Configuration, logging, error types, auth, middleware.  |

### Design rules

- Route handlers call services; services call the AI layer. Handlers never call
  Bedrock directly.
- All model access goes through a single client wrapper in `app/ai/`. Retry,
  timeout, and token-limit handling live there, and tests substitute it with a
  fake.
- Prompts are stored as versioned template files, not inline strings.
- Model output is parsed and validated against a schema before it is returned.
  Output that fails validation is retried or reported as an error, never passed
  through.
- Configuration is read from environment variables; no secrets are committed.

## API conventions

- Versioned under `/v1`.
- JSON request and response bodies with explicit schemas; OpenAPI is generated
  from the code.
- A `/health` endpoint for load balancer checks.
- Errors use a consistent envelope containing a machine-readable code, a
  message, and a request identifier.

## Cross-cutting concerns

- **Authentication:** requests must be authenticated at the service boundary.
  The mechanism is an open question (see below).
- **Observability:** structured JSON logs with a request identifier, plus
  metrics for latency, error rate, and model token usage.
- **Safety:** responses that involve health, exercise, or nutrition guidance
  pass through output checks and include appropriate disclaimers. Limits on
  recommended values are enforced in code rather than relying on the model.
- **Cost control:** per-request token ceilings and rate limiting.

## Open questions

- Initial set of AI capabilities and their request/response contracts.
- Authentication mechanism (Cognito-issued JWTs, API keys, or IAM-signed calls).
- Which Bedrock model(s) are used per capability.
- Whether any responses should be cached, and for how long.
