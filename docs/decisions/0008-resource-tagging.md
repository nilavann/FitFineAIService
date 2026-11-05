# 0008. Mandatory Project and Environment tags

**Status:** Accepted

## Context
Several environments share one or more AWS accounts and region. Without a
consistent label, resources are hard to find, audit, or attribute to cost.

## Decision
Every resource carries `Project=fitfine-ai-service` and
`Environment=<dev|staging|prod>`. The CDK app applies both tags at the top level
so all stacks inherit them, and a synthesis test enforces their presence.

## Alternatives considered
- **Tagging per stack or per resource:** easy to forget and inconsistent.
- **Naming conventions only:** helpful but not usable by billing or tag filters.

## Consequences
- Resources and costs can be filtered per project and environment.
- Some AWS resource types cannot be tagged; the test covers taggable types only.
- Cost allocation tags must be activated once in Billing before they appear in
  cost reports.
- Resources created by hand must be tagged manually.
