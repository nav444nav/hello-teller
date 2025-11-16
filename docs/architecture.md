# Teller Platform Architecture

## 1. Vision
We are building a teller-facing training platform that feels like a production bank's system while
staying inexpensive, easy to deploy, and safe. The entire stack is serverless on AWS so we only pay
for what we use. All infrastructure is described with AWS CDK, letting us deploy or destroy the
environment on demand.

## 2. High-level system overview
```
┌────────────┐       ┌────────────────────┐        ┌──────────────────┐
│ Angular UI │◀────▶│ API Gateway (HTTP) │◀──────▶│ Lambda Functions │
└────────────┘  WS  └────────────────────┘  events└──────────────────┘
      ▲  │                ▲         │                     │
      │  │                │         ▼                     ▼
      │  │       ┌────────────────────┐       ┌────────────────────┐
      │  └──────▶│ API Gateway (WS)   │◀────▶│ DynamoDB Tables    │
      │ updates  └────────────────────┘       └────────────────────┘
      │                                            ▲         │
      │                                            │         ▼
      │                                     ┌────────────────────┐
      │                                     │ EventBridge Bus    │
      │                                     └────────────────────┘
      │                                                │
      │                                     ┌────────────────────┐
      │                                     │ Step Functions     │
      │                                     └────────────────────┘
      ▼
┌────────────┐
│ Supervisors│ (same UI, elevated role)
└────────────┘
```

## 3. Repository layout
```
/README.md                 — product overview
/backend/                  — Lambda handlers, domain logic, shared libs
/frontend/teller-ui/       — Angular + NgRx SPA
/infra/                    — AWS CDK app (TypeScript) describing all stacks
/docs/architecture.md      — this architecture document
```

Each folder gets its own package.json to keep dependencies isolated. CDK stacks deploy the backend,
frontend hosting, and shared resources.

## 4. Backend architecture (AWS serverless)
### 4.1 API surface
* **REST (API Gateway HTTP API)**
  * `POST /sessions/open` — start a teller session with opening cash balance
  * `POST /sessions/{id}/close` — close a session, capturing closing balance
  * `POST /transactions` — create deposits or withdrawals
  * `POST /transactions/{id}/reverse` — request a reversal
  * `POST /approvals/{id}/decision` — supervisors approve/deny requests
  * `GET /dashboard` — fetch session summary, pending approvals, recent transactions (fallback when WS disconnects)
* **WebSocket (API Gateway WebSocket API)**
  * `$connect`, `$disconnect`, `$default` for lifecycle
  * Custom routes: `subscribeDashboard`, `unsubscribeDashboard` for real-time teller + supervisor feeds

### 4.2 Lambda functions (Node.js 18)
* `authHandler` — validates Amazon Cognito JWTs and injects identity context
* `sessionHandler` — handles open/close session requests
* `transactionHandler` — creates deposits/withdrawals, emits domain events
* `reversalWorkflowStarter` — initiates Step Function for reversals
* `approvalHandler` — records supervisor decisions
* `websocketConnectionHandler` — manages connections in DynamoDB
* `dashboardPublisher` — sends updates over WebSockets when domain events arrive
* `workflowCoordinator` — Step Functions task states executed as Lambda service integrations

### 4.3 Persistence (DynamoDB)
Provisioned as on-demand tables for cost control:
| Table | Primary key | Purpose |
|-------|-------------|---------|
| `TellerSessions` | PK: `sessionId`, SK: none | Tracks teller lifecycle, opening/closing balances |
| `Transactions` | PK: `transactionId` | Stores deposit/withdrawal and reversal info |
| `Approvals` | PK: `approvalId` | Stores approval tasks, links to transactions |
| `RealtimeConnections` | PK: `connectionId` | Maps WebSocket connections to teller/supervisor dashboards |

DynamoDB streams on `Transactions` and `Approvals` feed EventBridge for WebSocket updates and Step
Function triggers.

### 4.4 Eventing & workflows
* **EventBridge bus** for normalized events: `TransactionCreated`, `ApprovalRequested`,
  `ApprovalCompleted`, `SessionStateChanged`, `RealtimeBroadcastRequested`.
* **Step Functions** orchestrate multi-step flows:
  * **Deposit/withdrawal with approval** — check thresholds, wait for supervisor event, update status
  * **Reversal** — lock transaction, credit/debit reversal, notify teller, release lock
  * Step Functions are defined with the CDK using Amazon States Language inline JSON or the CDK
    builder API.
* **Event targets**: Lambda processors, Step Functions, and WebSocket broadcaster.

### 4.5 Authentication & authorization
* Amazon Cognito User Pools + Groups (Teller, Supervisor, Admin)
* Lambda authorizer attached to HTTP and WebSocket APIs
* Fine-grained checks in each handler enforce teller session ownership and supervisor roles.

## 5. Frontend architecture (Angular + NgRx)
* Angular 17 SPA deployed to S3 + CloudFront.
* Feature modules: `session`, `transactions`, `approvals`, `realtime`.
* NgRx store slices mirror backend resources (sessions, transactions, approvals, connection state).
* WebSocket service uses API Gateway WebSocket endpoint, dispatching `*_Received` actions.
* Angular Material for UI controls, `ngx-mask` for currency input, `rxjs` for reactive streams.
* Role-based access guards hide supervisor-only views while allowing same SPA for both personas.

### Key screens
1. **Teller console** — open/close session, create transactions, view history.
2. **Supervisor console** — approvals queue, transaction detail, approve/deny actions.
3. **Activity feed** — live stream of transaction status changes.

## 6. Infrastructure as Code (CDK)
The CDK app (TypeScript) defines multiple stacks:
1. `NetworkingStack` — optional VPC endpoints (API Gateway + DynamoDB are public so minimal).
2. `DataStack` — DynamoDB tables, EventBridge bus, IAM roles, KMS keys.
3. `BackendStack` — Lambda functions, API Gateway HTTP/WS APIs, Cognito pool.
4. `WorkflowStack` — Step Functions state machines.
5. `FrontendStack` — S3 bucket, CloudFront distribution, deployment pipeline (CodePipeline or GitHub
   Actions artifact).

Stacks share outputs via CDK `CfnOutput`s, and Stage definitions allow separate dev/demo envs. CDK
context stores threshold defaults and feature flags.

## 7. Domain workflows
### 7.1 Teller session lifecycle
1. Teller logs in (Cognito Hosted UI) and opens the SPA.
2. UI calls `POST /sessions/open` with starting cash.
3. Lambda writes session record and emits `SessionStateChanged` event.
4. Dashboard updates teller + supervisor views via WebSocket broadcast.
5. Closing session calculates totals and writes `SessionStateChanged(closed)`.

### 7.2 Cash transactions
1. Teller submits deposit/withdrawal specifying amount.
2. Lambda writes pending transaction.
3. If amount > threshold, Step Function waits on `ApprovalCompleted`; else auto-complete.
4. Completion event triggers UI broadcast and optionally prints receipt (future feature).

### 7.3 Supervisor approvals
1. `ApprovalRequested` event writes to `Approvals` table.
2. Supervisor UI fetches queue via WebSocket or REST fallback.
3. Decision writes result, triggers Step Function to continue workflow.

### 7.4 Reversals
1. Teller requests reversal referencing original transaction.
2. Step Function ensures session + amounts match, writes reversal transaction.
3. Updates propagate via EventBridge + WebSocket.

### 7.5 Real-time updates
* DynamoDB Streams → Lambda `dashboardPublisher`.
* Publisher resolves subscribers (connections filtered by teller/supervisor) and sends payloads
  through API Gateway WebSocket `@connections` endpoint.
* Frontend NgRx effects dispatch updates into the store.

## 8. Operations, security, and cost
* **Cost control:** on-demand DynamoDB, Lambda at low traffic, CloudFront caching for SPA.
* **Monitoring:** CloudWatch Logs Insights, metrics, alarms on error rates and throttles.
* **Tracing:** AWS X-Ray enabled on Lambda + API Gateway for debugging workflows.
* **Deployment:** CI pipeline runs `npm test`, `ng test`, `cdk synth`, then `cdk deploy --all` to a
  dev account. Manual approval for prod.
* **Disaster recovery:** Since this is educational, we rely on infrastructure-as-code redeployments
  plus periodic DynamoDB backups.

## 9. Future enhancements
* Swap DynamoDB-ledger for RDS/PostgreSQL ledger via Data API.
* Stream EventBridge events into S3 data lake using Firehose, query in Snowflake for analytics.
* Add teller scripting/training scenarios built on the same workflow engine.
* Support offline-first teller clients with IndexedDB caching.

## 10. Next steps
1. Scaffold CDK app with stacks outlined above.
2. Define DynamoDB tables + IAM roles.
3. Implement Lambda handlers + Step Function definitions.
4. Scaffold Angular workspace with NgRx store and WebSocket service.
5. Wire CI/CD to deploy backend + frontend artifacts.
