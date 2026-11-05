# FitFine AI Service

Backend service that exposes FitFine's AI capabilities over an HTTP API. It is
a standalone, API-only component: it has no user interface and is consumed by
other FitFine applications.

## Overview

| Aspect            | Choice                                          |
| ----------------- | ----------------------------------------------- |
| Language          | Python 3.12                                     |
| API framework     | FastAPI                                         |
| Model access      | Amazon Bedrock                                  |
| Compute           | AWS Lambda behind Amazon API Gateway (HTTP API) |
| Infrastructure    | AWS CDK (Python)                                |
| Packaging         | Lambda deployment package (zip) via CDK         |

## Documentation

- [Architecture](docs/architecture.md): components, request flow, and layering.
- [Deployment](docs/deployment.md): AWS topology, environments, and delivery pipeline.
- [Decision records](docs/decisions/): the reasoning behind significant choices.

## Repository layout (target)

```
app/            Application code (see docs/architecture.md)
infra/          AWS CDK stacks
tests/          Unit and integration tests
docs/           Design documentation
```

## Status

Design phase. The application skeleton has not yet been implemented.
