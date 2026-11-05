# Architecture

## Purpose and scope

The service provides AI-backed functionality to FitFine applications through
HTTP endpoints and events. It is a backend only. Member records and photos are
owned by Supabase; this service owns face embeddings and the logic that keeps
them in step with member photos.

## System context

```
Supabase (members, photos)
   │  database webhook (shared secret)
   ▼
API Gateway ──▶ API Lambda (FastAPI) ──▶ SQS queue ──▶ Worker Lambda ──▶ Rekognition
   ▲                  │                    └─▶ DLQ          │
   │ Supabase token   └──▶ Rekognition (search)              └──▶ Supabase (photo, status)
Staff app

CloudWatch collects logs and metrics from both functions.
```

The functions are not attached to a VPC. They reach AWS services and Supabase
over TLS, authenticated by IAM roles and stored credentials respectively.

## Components

| Component     | Responsibility                                                          |
| ------------- | ----------------------------------------------------------------------- |
| API Lambda    | FastAPI app (via Mangum): webhook intake and photo search.              |
| Worker Lambda | Consumes queue jobs; keeps Rekognition in step with member photos.      |
| SQS + DLQ     | Decouples intake from processing; provides retries and failure parking. |
| Rekognition   | Face collection holding embeddings, keyed by member identifier.         |
| Supabase      | Source of truth for members and photos; receives enrolment status.      |

## Capabilities

- [Member face recognition](features/member-face-recognition.md): enrol member
  photos from Supabase events and find members from a photo.

## Layering

Code is organised so each layer depends only on the one beneath it.

| Layer       | Location          | Responsibility                                            |
| ----------- | ----------------- | --------------------------------------------------------- |
| Entry points| `app/api/`, `app/worker/` | HTTP routes and queue handler. No business logic. |
| Schemas     | `app/schemas/`    | Typed request, response, and event models (Pydantic).     |
| Services    | `app/services/`   | Feature logic such as enrol, remove, and search.          |
| Integrations| `app/integrations/` | Thin clients for Rekognition, Supabase, and SQS.        |
| Core        | `app/core/`       | Configuration, logging, errors, auth, middleware.         |

### Design rules

- Entry points call services; services call integrations. Entry points never
  call AWS or Supabase directly.
- Every external dependency sits behind one integration wrapper that owns
  timeouts and retries, and that tests replace with a fake.
- Events carry references (identifiers and storage paths), never image bytes.
  Workers re-read current state before acting, so stale or duplicate events
  are harmless.
- Handlers are idempotent: processing the same job twice leaves the same result.
- Configuration comes from environment variables and SSM parameters; no secrets
  are committed.

## API conventions

- Versioned under `/v1`; JSON bodies with explicit schemas; OpenAPI generated
  from the code.
- `/health` endpoint for monitoring.
- Errors use one envelope: machine-readable code, message, request identifier.

## Authentication

| Caller              | Mechanism                                                       |
| ------------------- | --------------------------------------------------------------- |
| Staff app (search)  | Supabase access token, verified by the API Lambda against the project's JWT secret. Only staff hold accounts in the project, so any valid token is accepted; no role check. |
| Supabase (webhook)  | Shared secret sent in a request header.                         |
| Service to AWS      | IAM execution roles with least privilege.                       |
| Service to Supabase | Service key held in SSM Parameter Store.                        |

Token verification lives in one function in `app/core/auth.py`, so moving from
the legacy shared secret to Supabase's asymmetric signing keys later is a local
change (see ADR 0007).

## Cross-cutting concerns

- **Observability:** structured JSON logs with a request identifier; metrics for
  latency, errors, queue depth, and dead-letter count; alarms on the last two.
- **Safety and privacy:** face data is biometric data; see the feature document.
- **Cost control:** API Gateway throttling, bounded worker concurrency, and no
  always-on resources.
- **Latency:** the API integration timeout is 30 seconds. Search completes well
  within this; enrolment runs asynchronously and is not bound by it.

## Open questions

- Contracts for capabilities beyond face recognition.
