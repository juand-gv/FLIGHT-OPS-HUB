# Flight Ops Disruption Hub — Claude Context

## Always Read First
- [README.md](README.md) — Project overview, architecture, event model, agent strategy, and CDK structure
- [tasks.md](tasks.md) — Phased implementation task spec; check current progress and open tasks before starting any work

## Project Summary
Event-driven airline disruption management system.
Stack: Python · AWS CDK · EventBridge · SQS · DynamoDB · Lambda
Goal: Stay within AWS Free Tier while applying enterprise-grade patterns.

## Key Constraints
- No VPC, no NAT Gateway for MVP
- DynamoDB on-demand capacity only
- CloudWatch log retention: 7 days max
- HTTP API Gateway (not REST API)
- No provisioned Lambda concurrency

## Code Conventions
- All Lambda handlers use AWS Lambda Powertools (structured logging, tracing, idempotency)
- Events follow the versioned envelope schema defined in README.md
- Correlation IDs must be propagated end-to-end
- All consumers must be idempotent
- Every SQS queue must have a DLQ
- **Before writing or reviewing any Python or CDK code, read [standards.md](standards.md)**

## CDK Conventions
- One stack per concern: api_stack, event_bus_stack, consumer_stack, data_stack
- Reusable patterns live in constructs/
- Lambda handler code lives in services/<domain>/

## Testing
- Unit tests in tests/unit/
- Integration tests in tests/integration/
