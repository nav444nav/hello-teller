# Hello Teller

A serverless teller platform that simulates a production bank's teller console while staying safe,
small, and inexpensive. The goal is to teach modern teller workflows, approvals, and real-time UI
patterns on AWS.

## Guiding principles
* **Serverless-first:** API Gateway, Lambda, DynamoDB, EventBridge, and Step Functions handle all
  backend logic so the platform costs almost nothing when idle.
* **Realistic UX:** Angular + NgRx SPA mirrors the teller/supervisor consoles used in real banks,
  including session management, approvals, and reversals.
* **Infrastructure as code:** AWS CDK provisions every resource (backend, workflows, hosting) so we
  can deploy or destroy the full environment quickly.

## Core features
* Teller login, session open/close with enforced balances.
* Cash transactions (deposit, withdrawal) with configurable approval thresholds.
* Supervisor approvals surfaced as a queue in the UI, with audit history.
* Reversals modeled as first-class transactions with their own workflows.
* Real-time UI updates via API Gateway WebSockets so tellers and supervisors always see the latest
  data.
* Event-driven backend using EventBridge + Step Functions for multi-step transaction flows.

## Architecture highlights
* **Backend:** API Gateway (HTTP + WebSocket) → Lambda → DynamoDB, with EventBridge and Step
  Functions orchestrating approvals and reversals.
* **Frontend:** Angular + NgRx SPA hosted in S3 + CloudFront, sharing one codebase for tellers and
  supervisors using role-based guards.
* **Operations:** CloudWatch + X-Ray monitoring, Cognito authentication, CI/CD pipeline running
  tests, `cdk synth`, and `cdk deploy`.

## Implementation plan & cost guardrail
We want to keep monthly spend under **$25** by relying on serverless, on-demand resources and
tearing down demo stacks when idle. Delivery proceeds in two waves:

1. **Backend-first:** build the CDK stacks, DynamoDB tables, Lambda handlers, and API Gateway
   endpoints, then exercise them via Postman/newman collections before any UI work.
2. **Frontend enablement:** once backend flows are verified, scaffold the Angular + NgRx SPA,
   connect to the WebSocket feed, and deploy to S3 + CloudFront.

Detailed TODOs for each phase live in [`docs/todo.md`](docs/todo.md).

See [`docs/architecture.md`](docs/architecture.md) for detailed diagrams, workflows, and next steps.

## Future-ready
Although this is not a production core ledger, the design anticipates future upgrades:
* Swap in Amazon RDS or Aurora PostgreSQL for a double-entry journal database.
* Stream operational data to S3 for a data lake, then analyze in Snowflake or Athena.
* Extend workflows with teller training scenarios or offline-capable clients.
