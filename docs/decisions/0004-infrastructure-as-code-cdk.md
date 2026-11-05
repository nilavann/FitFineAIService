# 0004. AWS CDK (Python) for infrastructure

**Status:** Accepted

## Decision
Define infrastructure with AWS CDK in Python, kept in `infra/` alongside the
application.

## Consequences
- One language across application and infrastructure.
- Infrastructure changes are reviewed and versioned with the code.
- Requires CloudFormation bootstrap in each target account and region.
