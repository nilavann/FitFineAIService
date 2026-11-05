# 0002. Amazon Bedrock for model access

**Status:** Accepted

## Context
The service is deployed on AWS and needs hosted foundation models without
operating inference infrastructure.

## Decision
Invoke models through Amazon Bedrock, authenticated by the function's IAM role.
All calls go through one client wrapper in `app/ai/`.

## Consequences
- No API keys to store or rotate for model access; access is governed by IAM.
- Usage and billing consolidate in AWS.
- Available models and quotas are bound by Bedrock's per-region offering.
- The wrapper isolates the rest of the code from provider-specific details,
  keeping a future model change local.
