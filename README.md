# FitFine AI Service

Backend service that exposes FitFine's AI capabilities over an HTTP API. It is
a standalone, API-only component: it has no user interface and is consumed by
other FitFine applications.

## Overview

| Aspect            | Choice                                          |
| ----------------- | ----------------------------------------------- |
| Language          | Python 3.12                                     |
| API framework     | FastAPI                                         |
| Face recognition  | Amazon Rekognition (face collections)           |
| Member data       | Supabase (source of truth, external)            |
| Async processing  | Amazon SQS with a dead-letter queue             |
| Generative models | Amazon Bedrock (reserved for later features)    |
| Compute           | AWS Lambda (API and worker) behind API Gateway (HTTP API) |
| Infrastructure    | AWS CDK (Python)                                |
| Packaging         | Lambda deployment package (zip) via CDK         |

## Documentation

- [Architecture](docs/architecture.md): components, request flow, and layering.
- [Deployment](docs/deployment.md): AWS topology, environments, and delivery pipeline.
- [Member face recognition](docs/features/member-face-recognition.md): enrolment from Supabase events, and search by photo.
- [Decision records](docs/decisions/): the reasoning behind significant choices.

## Repository layout (target)

```
app/            Application code: api, worker, services, integrations
infra/          AWS CDK stacks
tests/          Unit and integration tests
docs/           Design documentation
```

## Status

Design phase. The application skeleton has not yet been implemented.
