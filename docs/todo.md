# Delivery Plan & TODOs

## Cost guardrail: $25/month target
* **DynamoDB on-demand (2 tables with ~1M RUs/month)** — ~$6
* **Lambda + API Gateway (light training usage)** — ~$8
* **EventBridge + Step Functions (sporadic)** — ~$3
* **Cognito + CloudWatch + misc.** — ~$3
* **S3 + CloudFront for SPA** — ~$5

> **Action:** keep provisioned capacity off, rely on AWS Free Tier in dev, and delete demo stacks when not needed.

## Immediate next step — scaffold the CDK app
1. Install the AWS CDK toolchain (`npm install -g aws-cdk`) and run `cdk init app --language typescript` inside `infra/` so we can synthesize stacks locally before wiring any services.
2. Create placeholder stacks (`DataStack`, `BackendStack`, `WorkflowStack`) plus a `PipelineStage` construct that composes them, matching the bullets below.
3. Add a `cdk.json` context entry (e.g., `{ "app": "npx ts-node bin/hello-teller.ts" }`) and a `.env.example` documenting the AWS profile/region expected when running `cdk synth`.
4. Validate the empty stacks with `npm run build && cdk synth` to make sure the baseline project compiles; this gives us a concrete place to start adding DynamoDB tables and Lambdas.

## Phase 1 — Backend-first (test via Postman)
- [ ] Scaffold CDK app with `DataStack`, `BackendStack`, and `WorkflowStack` plus dev/prod stages.
- [ ] Define DynamoDB tables (Sessions, Transactions, Approvals, Connections) with on-demand billing.
- [ ] Implement Lambda handlers for sessions, transactions, approvals, reversals, and WebSocket lifecycle.
- [ ] Configure API Gateway HTTP + WebSocket APIs with Cognito authorizer.
- [ ] Wire DynamoDB Streams → EventBridge → Step Functions + `dashboardPublisher`.
- [ ] Author Postman collection covering login (Cognito hosted UI token), session open/close, txn create, approvals, reversals.
- [ ] Enable CloudWatch dashboards + alarms focused on throttles and error counts.

## Phase 2 — Frontend enablement
- [ ] Scaffold Angular workspace with NgRx store slices (sessions, transactions, approvals, realtime).
- [ ] Implement WebSocket service and NgRx effects that mirror EventBridge notifications.
- [ ] Build teller console screens (session management, txn form, activity feed).
- [ ] Build supervisor console screens (approval queue, decision modals).
- [ ] Integrate Cognito auth flow + role-based routing guards.
- [ ] Publish SPA to S3 + CloudFront via CDK pipeline.

## Operational hygiene
- [ ] Document teardown procedure (`cdk destroy --all`) to avoid idle spend.
- [ ] Schedule monthly cost review + AWS Cost Anomaly Detection alert at $20/mo.
- [ ] Add integration tests (via Postman CLI/newman) to CI to keep backend verified before UI deploys.
