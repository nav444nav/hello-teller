# hello-teller
We are building a teller banking platform as a realistic but safe learning project.

Backend

Runs entirely on AWS serverless (API Gateway HTTP + WebSocket, Lambda, DynamoDB, EventBridge, Step Functions).

Supports:

Teller sessions (open/close with balances)

Cash transactions (deposit, withdrawal)

Supervisor approvals for high-value transactions

Transaction reversals as separate, linked transactions

Uses EventBridge for domain events (e.g. TransactionUpdated, ApprovalRequested, TellerSessionOpened) and Step Functions for multi-step workflows (deposits and reversals with approvals).

Stores operational data in DynamoDB (Transactions, Approvals, TellerSessions, RealtimeConnections), with the design allowing a future RDS “journal” DB if needed.

Frontend

A single-page Angular + NgRx UI that:

Lets tellers open/close sessions

Create deposits/withdrawals

Request reversals

Shows supervisors a list of pending approvals

Receives real-time updates over WebSockets (e.g., transaction status changes, new approvals)

Hosted on AWS via S3 + CloudFront.

Architecture & goals

Monorepo: infra/ for all CDK infrastructure, backend/ for domain and Lambda code, frontend/teller-ui/ for the Angular app.

Realistic patterns: event-driven, real-time, approvals, reversals, workflows.

All infra is defined in CDK so the whole system can be deployed and destroyed easily for cost control.

Target cost: within $25–50/month, mostly idle and used via Postman + the UI for learning and demos.
