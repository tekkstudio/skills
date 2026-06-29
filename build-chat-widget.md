# /build-chat-widget

You are a senior AWS serverless architect. Your job is to generate production-ready infrastructure
code and application scaffolding for a multi-tenant AI chat widget — including AppSync, Lambda,
DynamoDB, and Bedrock Knowledge Base — with optional SQS async workers, WAF, and ECS Fargate
crawler depending on scope.

The two-plane design — control plane (JWT issuance) + execution plane (AppSync + Lambda + Bedrock)
— is the canonical pattern. Generate code that matches it exactly.

---

## Step 1 — Detect stack and scope

Parse $ARGUMENTS for:
- IaC stack: `terraform`, `sam`, `cdk` (default: ask if not provided)
- Scope: `core`, `full`, `appsync`, `dynamodb`, `lambda`, `auth` (default: ask if not provided)
- Lambda language: `python` (default), `typescript`
- Project name prefix (default: `chat-widget`)
- AWS region (default: `us-east-1`)

If $ARGUMENTS is empty, ask:
1. Which IaC stack? (SAM recommended for beginners / Terraform / CDK TypeScript)
2. Scope:
   - **core** (recommended) — AppSync + Lambda authorizer + ChatApiFunction + DynamoDB + Bedrock KB.
     The complete real-time chat layer. No WAF, no SQS workers, no containers.
     Estimated cost at low volume: ~$0.50–$1/month (Bedrock inference only).
     Start here.
   - **full** — everything in core, plus WAF (~$5/month fixed), SQS async workers (handoff
     scoring, session analysis, integrations), and ECS Fargate web crawler.
     Choose this when you are ready for production hardening and automated KB ingestion.
     Significantly higher operational complexity and fixed infrastructure cost.
   - Or name a specific layer: `appsync`, `dynamodb`, `lambda`, `auth`
3. Lambda language: python or typescript?
4. Project name prefix?
5. AWS region?

---

## Step 2 — Scaffold project structure

Create the directory structure before writing any files.

**SAM layout (core scope):**
```
{project_name}/
├── infra/main.yml                       # stateful layer — DynamoDB, S3, Bedrock KB, KMS
├── backend/templates/sam_template.yml   # compute layer — Lambda, AppSync
├── backend/src/
│   ├── authorizer/
│   │   ├── handler.py        # entry point only — parse event, call service, return policy
│   │   └── domain.py         # pure: validate_origin(), build_policy()
│   ├── session_token/
│   │   ├── handler.py        # entry point only
│   │   └── domain.py         # pure: validate_origin(), build_jwt_claims(), build_cors_headers()
│   ├── chat_api/
│   │   ├── handler.py        # entry point only
│   │   ├── services/
│   │   │   ├── quota.py      # check_quota()
│   │   │   ├── session.py    # load_history(), append_messages()
│   │   │   └── bedrock.py    # retrieve_kb_context(), invoke_model()
│   │   ├── domain.py         # pure: build_system_prompt(), format_messages()
│   │   └── adapters/
│   │       ├── dynamo.py
│   │       ├── bedrock_client.py
│   │       └── sqs.py        # conditional publish
│   └── shared/               # jwt_utils.py, models.py
├── schema/schema.graphql
├── widget/
│   ├── auth.js          # token fetch + refresh — nothing else
│   ├── connection.js    # AppSync WebSocket lifecycle — nothing else
│   ├── widget.js        # public API (ChatWidget) — thin coordinator only
│   ├── config.json      # hot-swap: AppSync endpoints without bundle redeploy
│   └── demo/
│       └── index.html   # test page — loads widget.js, owns the UI
├── scripts/
│   ├── update-config.sh # reads compute stack outputs → writes widget/config.json
│   └── seed-tenant.sh   # seeds test CONFIG + subscription records in DynamoDB
└── .env.example
```

**SAM layout (full scope — adds WAF, SQS workers, ECS crawler):**
```
{project_name}/
├── infra/main.yml
├── backend/templates/sam_template.yml
├── backend/src/
│   ├── authorizer/
│   │   ├── handler.py
│   │   └── domain.py
│   ├── session_token/
│   │   ├── handler.py
│   │   └── domain.py
│   ├── chat_api/
│   │   ├── handler.py
│   │   ├── services/
│   │   │   ├── quota.py
│   │   │   ├── session.py
│   │   │   └── bedrock.py
│   │   ├── domain.py
│   │   └── adapters/
│   │       ├── dynamo.py
│   │       ├── bedrock_client.py
│   │       └── sqs.py
│   ├── handoff_scoring/handler.py
│   ├── session_analysis/handler.py
│   ├── integrations_worker/handler.py
│   └── shared/
├── crawler/
│   ├── Dockerfile
│   ├── crawler.py
│   └── requirements.txt
├── schema/schema.graphql
├── widget/
│   ├── auth.js
│   ├── connection.js
│   ├── widget.js
│   ├── config.json
│   └── demo/
│       └── index.html
├── scripts/
│   ├── update-config.sh
│   └── seed-tenant.sh
└── .env.example
```

**Terraform layout:** mirror the SAM layout above, replacing `infra/main.yml` and
`backend/templates/sam_template.yml` with `terraform/modules/` (dynamodb/, appsync/,
lambda/, and for full scope: sqs/, waf/, ecs/).

**CDK layout:** mirror the SAM layout above, splitting into `lib/stacks/stateful-stack.ts`
(DynamoDB, S3, Bedrock KB) and `lib/stacks/compute-stack.ts` (Lambda, AppSync, and for
full scope: SQS, WAF, ECS).

---

## Step 3 — Generate all files completely

Write every file with complete, working code. No placeholders, no TODOs, no stub functions.

### schema/schema.graphql — generate first, it is the contract

```graphql
type Message {
  messageId: ID!
  tenantId: String!
  sessionId: String!
  role: String!
  content: String!
  timestamp: AWSDateTime!
}
type SendMessageResponse { messageId: ID!, reply: String!, sessionId: String! }
type LeadSubmitResponse  { leadId: ID!, success: Boolean! }
type WidgetConfig {
  agentName: String, primaryColor: String, secondaryColor: String,
  avatarUrl: String, placeholder: String
}
type Query {
  getSessionMessages(tenantId: String!, sessionId: String!, limit: Int): [Message]
  getWidgetConfig(tenantId: String!): WidgetConfig
}
type Mutation {
  sendMessage(tenantId: String!, sessionId: String!, message: String!,
    visitorId: String): SendMessageResponse
  submitLead(tenantId: String!, sessionId: String!, email: String!,
    name: String): LeadSubmitResponse
}
```

### DynamoDB tables — use these exact key patterns

**ChatSessionTable**
- PK: `TENANT#{tenant_id}` · SK: `CONFIG` | `SESSION#{session_id}` | `LEAD#{lead_id}`
- BillingMode: `PAY_PER_REQUEST`
- GSIs (all with hash=PK unless noted):
  - `UpdatedAtIndex` — range=`updatedAt`, projection=`ALL`
  - `MessageCountIndex` — range=`messageCount`, projection=`ALL`
  - `HandoffStatusIndex` — range=`handoffStatus`, projection=`INCLUDE` relevant session metadata
  - `LeadReadStatusIndex` — range=read-status field, projection=`KEYS_ONLY`

**Message storage pattern:** Store messages as a list attribute on the session item — one DynamoDB
item per session, not one item per message. Append new messages atomically with a conditional
UpdateExpression to avoid race conditions.

**VisitorProfileTable**
- PK: `TENANT#{tenant_id}` · SK: `VISITOR#{visitor_id}` | `SESSION#{session_id}`
- PITR: enabled
- GSIs covering three access patterns: by tenant+session, by email, by tenant profile.
  Choose attribute naming that makes these key conditions explicit in your item writes.

**TokenUsageTable**
- PK: `TenantID` · SK: `Timestamp`
- GSIs:
  - `YearMonthIndex` — hash=`YearMonth`, range=`Timestamp`, projection=`INCLUDE` billing fields
  - `RecordTypeIndex` — hash=`RecordType`, range=`YearMonth`

**SubscriptionTable**
- PK: `SUBSCRIPTION#{stripe_subscription_id}` · SK: `METADATA`
- PITR: enabled
- GSIs covering: lookup by tenant_id (`ByTenant` — the primary runtime check), by Stripe
  customer ID, and by internal user ID.

### infra/main.yml — complete stateful layer (SAM / CloudFormation)

Generate all stateful resources in one stack so a single `sam deploy` bootstraps the entire
environment with no manual AWS console steps. Include these resources in this order:

**1. JWT Secret — auto-generated, no manual step**
```yaml
JwtSecret:
  Type: AWS::SecretsManager::Secret
  Properties:
    Name: !Sub /${ProjectName}/${Environment}/jwt-secret
    GenerateSecretString:
      PasswordLength: 64
      ExcludeCharacters: '"@/\'
```
No `openssl` command needed. The secret value is generated on first deploy and rotatable
without touching the compute stack.

**2. SSM Parameters — bridge between infra and compute stacks**
Write the JWT secret ARN and (later) the KB ID to SSM so the compute stack can reference
them without hardcoding or cross-stack exports:
```yaml
JwtSecretArnParam:
  Type: AWS::SSM::Parameter
  Properties:
    Name: !Sub /${ProjectName}/${Environment}/jwt-secret-arn
    Value: !Ref JwtSecret
    Type: String

KnowledgeBaseIdParam:
  Type: AWS::SSM::Parameter
  Properties:
    Name: !Sub /${ProjectName}/${Environment}/knowledge-base-id
    Value: !Ref BedrockKnowledgeBase
    Type: String
```

**3. DynamoDB tables** — as specified above (ChatSessionTable, VisitorProfileTable,
TokenUsageTable, SubscriptionTable).

**4. S3 bucket** — for KB document storage. Versioning enabled. Block all public access.
Prefix convention: `documents/{tenant_id}/` for KB source files.

**5. OpenSearch Serverless** — Bedrock requires a vector store. Deploy in this exact order
(each resource depends on the previous):

```
a. EncryptionSecurityPolicy  (Type: encryption)
   → Policy: AWSOwnedKey on the collection

b. NetworkSecurityPolicy  (Type: network)
   → AllowFromPublic: true  (Bedrock service accesses it; VPC endpoint is optional)

c. OpenSearchServerlessCollection  (Type: VECTORSEARCH)
   → DependsOn: EncryptionSecurityPolicy, NetworkSecurityPolicy

d. BedrockKBRole  (IAM role, AssumeRolePrincipal: bedrock.amazonaws.com)
   → Policies: s3:GetObject + s3:ListBucket on DataBucket, aoss:APIAccessAll on collection

e. DataAccessPolicy  (Type: data)
   → Grants BedrockKBRole: aoss:CreateIndex, DescribeIndex, ReadDocument, WriteDocument
      on index/{collection-name}/*
   → DependsOn: BedrockKBRole, OpenSearchServerlessCollection
```

**6. Bedrock Knowledge Base — one per deployment, shared by all tenants**
`core` scope uses a single shared KB. Every tenant's ChatApiFunction reads from the same
knowledge base ID. This is intentional — do not create per-tenant KBs.

```yaml
BedrockKnowledgeBase:
  Type: AWS::Bedrock::KnowledgeBase
  DependsOn: DataAccessPolicy
  Properties:
    Name: !Sub ${ProjectName}-${Environment}-kb
    RoleArn: !GetAtt BedrockKBRole.Arn
    KnowledgeBaseConfiguration:
      Type: VECTOR
      VectorKnowledgeBaseConfiguration:
        EmbeddingModelArn: !Sub arn:aws:bedrock:${AWS::Region}::foundation-model/amazon.titan-embed-text-v2:0
    StorageConfiguration:
      Type: OPENSEARCH_SERVERLESS
      OpensearchServerlessConfiguration:
        CollectionArn: !GetAtt OpenSearchServerlessCollection.Arn
        VectorIndexName: bedrock-knowledge-base-default-index
        FieldMapping:
          VectorField: bedrock-knowledge-base-default-vector
          TextField: AMAZON_BEDROCK_TEXT_CHUNK
          MetadataField: AMAZON_BEDROCK_METADATA
```

**7. Bedrock Data Source** — points to the S3 bucket, documents/ prefix:
```yaml
BedrockDataSource:
  Type: AWS::Bedrock::DataSource
  Properties:
    KnowledgeBaseId: !Ref BedrockKnowledgeBase
    Name: !Sub ${ProjectName}-${Environment}-datasource
    DataSourceConfiguration:
      Type: S3
      S3Configuration:
        BucketArn: !GetAtt DataBucket.Arn
        InclusionPrefixes:
          - documents/
```

**Stack Outputs** — the compute stack reads these via SSM; also export for reference:
```yaml
Outputs:
  KnowledgeBaseId:   { Value: !Ref BedrockKnowledgeBase }
  DataSourceId:      { Value: !GetAtt BedrockDataSource.DataSourceId }
  JwtSecretArn:      { Value: !Ref JwtSecret }
  ChatSessionTable:  { Value: !Ref ChatSessionTable }
  SubscriptionTable: { Value: !Ref SubscriptionTable }
  TokenUsageTable:   { Value: !Ref TokenUsageTable }
  DataBucketName:    { Value: !Ref DataBucket }
```

### scripts/update-config.sh — auto-populate widget/config.json

Generate a complete executable shell script:
```bash
#!/usr/bin/env bash
# Usage: ./scripts/update-config.sh <compute-stack-name> [region]
STACK=${1:?Usage: update-config.sh <compute-stack-name> [region]}
REGION=${2:-us-east-1}

get_output() {
  aws cloudformation describe-stacks \
    --stack-name "$STACK" --region "$REGION" \
    --query "Stacks[0].Outputs[?OutputKey=='$1'].OutputValue" \
    --output text
}

GRAPHQL=$(get_output AppSyncGraphQLUrl)
REALTIME=$(get_output AppSyncRealtimeUrl)
SESSION=$(get_output SessionTokenApiUrl)

cat > widget/config.json <<EOF
{
  "graphqlUrl":      "$GRAPHQL",
  "realtimeUrl":     "$REALTIME",
  "sessionTokenUrl": "$SESSION"
}
EOF
echo "✅ widget/config.json updated"
```

### scripts/seed-tenant.sh — seed test records

Generate a complete executable shell script that seeds both the CONFIG record and a dummy
subscription so `SessionTokenFunction` will issue a token on first request:
```bash
#!/usr/bin/env bash
# Usage: ./scripts/seed-tenant.sh <infra-stack-name> [tenant-id] [region]
STACK=${1:?Usage: seed-tenant.sh <infra-stack-name> [tenant-id] [region]}
TENANT=${2:-test-tenant-id}
REGION=${3:-us-east-1}

CHAT_TABLE=$(aws cloudformation describe-stacks --stack-name "$STACK" --region "$REGION" \
  --query "Stacks[0].Outputs[?OutputKey=='ChatSessionTable'].OutputValue" --output text)
SUB_TABLE=$(aws cloudformation describe-stacks --stack-name "$STACK" --region "$REGION" \
  --query "Stacks[0].Outputs[?OutputKey=='SubscriptionTable'].OutputValue" --output text)

aws dynamodb put-item --table-name "$CHAT_TABLE" --region "$REGION" --item '{
  "PK":             {"S": "TENANT#'"$TENANT"'"},
  "SK":             {"S": "CONFIG"},
  "allowedDomains": {"L": [{"S": "localhost"}]},
  "systemPrompt":   {"S": "You are a helpful assistant."},
  "planConfig":     {"M": {"maxMessagesPerMonth": {"N": "1000"}}},
  "isArchived":     {"BOOL": false},
  "userSuspended":  {"BOOL": false}
}'

aws dynamodb put-item --table-name "$SUB_TABLE" --region "$REGION" --item '{
  "PK":       {"S": "SUBSCRIPTION#test-sub-'"$TENANT"'"},
  "SK":       {"S": "METADATA"},
  "tenantId": {"S": "'"$TENANT"'"},
  "status":   {"S": "active"}
}'

echo "✅ Seeded tenant $TENANT in $CHAT_TABLE and $SUB_TABLE"
```

### Backend layer rules — apply to every Lambda

These rules apply to all generated backend code. Enforce them before writing any handler.

**Layer direction: entry point → services → domain. Never reverse.**
- `handler.py` is the entry point only. Parse the event, call a service function, return
  the response. No business logic, no boto3 calls, no conditional branching beyond input
  validation. If `handler.py` exceeds ~30 lines it is doing too much.
- Services orchestrate: they call adapters and domain functions, never each other recursively.
- Domain functions are pure: deterministic output, no side effects, no AWS SDK imports.
  `validate_origin()`, `build_policy()`, `build_system_prompt()`, `format_messages()` —
  all pure. A pure function can be called twice with the same args and return the same result.
- Adapters are the only boto3 boundary. `dynamo.py`, `bedrock_client.py`, `sqs.py` import
  `boto3`. Nothing outside `adapters/` does.

**Read paths must never write.**
`get_tenant_config()`, `load_history()`, `check_quota()` — these functions read. They must
not update a counter, set a timestamp, or write any attribute as a side effect. Any mutation
belongs in an explicit write function called separately by the service layer.

**Module-scope caching for AWS clients and secrets.**
Initialise `boto3` clients and SecretsManager values once at module scope, not inside the
handler function. Lambda reuses the execution environment across warm invocations — per-request
initialisation is unnecessary latency.

```python
# Correct — module scope
_dynamodb = boto3.resource('dynamodb')
_table    = _dynamodb.Table(os.environ['CHAT_TABLE_NAME'])
_secret   = None  # lazy-loaded on first call, then cached

# Wrong — inside handler
def lambda_handler(event, context):
    dynamodb = boto3.resource('dynamodb')  # re-initialised on every call
```

### Lambda: authorizer/handler.py — full implementation

- Import: `boto3`, `jwt`, `re`, `os`
- Cache JWT secret from SecretsManager in **module scope** (avoid per-request fetches)
- `jwt.decode`: algorithm HS256, audience=`"chat-api"`, issuer=`"chat-widget-auth"`
- Get TENANT CONFIG item from DynamoDB (`PK=TENANT#{tenant_id}`, `SK=CONFIG`) — use `.get()`
  safe access throughout, never direct key access that raises KeyError
- Return Deny if config missing, tenant archived, or tenant suspended
- `validate_origin`: strip scheme + www, check exact match or subdomain suffix against `allowedDomains`
- Return Allow policy with `context={"tenantId": tenant_id}` or Deny policy
- `build_policy` helper returns `{"principalId", "policyDocument": {"Version", "Statement"}}`

### Lambda: session_token/handler.py — full implementation

This is the only REST endpoint in the architecture. The widget calls it on startup to get a JWT
before opening the AppSync WebSocket. It is the control-plane equivalent for readers who do not
have a separate backend service.

- HTTP API Gateway (not REST API): `POST /session`, OPTIONS for CORS preflight
- Request body: `{ "tenantId": "..." }`
- Read `Origin` header (case-insensitive — HTTP API Gateway normalises headers to lowercase)
- Load CONFIG from DynamoDB (`PK=TENANT#{tenant_id}`, `SK=CONFIG`) using `.get()` throughout —
  never direct key access
- Return 403 if config missing, `isArchived=true`, or `userSuspended=true`
- `validate_origin`: strip scheme + www from origin, check exact match or subdomain suffix
  against each entry in `allowedDomains`
- Return 403 if origin fails validation
- Query the `ByTenant` GSI on SubscriptionTable for an active subscription record
  (check `status` is in `active`, `trialing`, or your admin-override equivalent)
- Return 403 if no active subscription found
- Load JWT secret from SecretsManager — cache at **module scope**, same pattern as the authorizer
- Issue HS256 JWT with claims: `iss="chat-widget-auth"`, `aud="chat-api"`, `exp=now+30min`,
  `tenantId` in payload — these exact values must match what the authorizer expects
- Return 200: `{ "token": "..." }`
- CORS response header: `Access-Control-Allow-Origin` set to the validated request origin —
  never wildcard (`*`) on a successful 200 response

**This function uses the same DynamoDB tables and JWT secret as the Lambda Authorizer.**
The authorizer validates what this function issues — keep `iss`, `aud`, and algorithm in sync.
If they diverge, every widget request will be denied.

### Lambda: chat_api/ — full implementation (layered)

`handler.py` — entry point only:
```python
# All business logic lives in services/. handler.py only parses the event and routes.
identity       = event.get('identity', {})
resolver_ctx   = identity.get('resolverContext', {})
tenant_id      = resolver_ctx.get('tenantId') or arguments.get('tenantId')
# → call chat_service.handle(tenant_id, session_id, message, visitor_id)
# → return {messageId, reply, sessionId}
```

`services/quota.py` — `check_quota(tenant_id, plan_config)`:
- Query `TokenUsageTable` `YearMonthIndex` for current month usage
- Return `(allowed: bool, current: int, limit: int)`
- Read only — never writes a counter

`services/session.py` — two explicit functions, never combined:
- `load_history(tenant_id, session_id)` → returns message list. Read only.
- `append_messages(tenant_id, session_id, user_msg, assistant_msg)` → atomic update.
  Called by the service layer after the model responds, never inside a read path.

`services/bedrock.py`:
- `retrieve_kb_context(kb_id, query)` → returns list of text passages, degrades to `[]`
  on any exception (KB unavailable must never break chat)
- `invoke_model(system_prompt, messages)` → invokes `us.amazon.nova-2-lite-v1:0` via
  Strands Agent (`BedrockModel` + `Agent(model, system_prompt)`), returns reply string

`domain.py` — pure functions only:
- `build_system_prompt(base_prompt, kb_passages)` → string. No I/O.
- `format_messages(history)` → list formatted for Bedrock converse API. No I/O.
- `new_message_id()` → returns a UUID string. Acceptable impurity — determinism not required here.

`adapters/sqs.py` — conditional publish (`full` scope):
```python
def publish_if_configured(queue_url_env, payload):
    url = os.environ.get(queue_url_env)
    if url:
        sqs.send_message(QueueUrl=url, MessageBody=json.dumps(payload))
```

Log token usage and latency to `TokenUsageTable` after the model responds.

### Lambda: handoff_scoring/handler.py and session_analysis/handler.py (`full` scope only)

Skip these files entirely when scope is `core`.

Both async workers use **Nova Micro** (`amazon.nova-micro-v1:0`), not Nova 2 Lite — these are
classification and summarisation tasks where cost matters more than response richness.
Generate complete handlers using the Bedrock `converse` API.

### AppSync API resource (all stacks)

- `AuthenticationType: AWS_LAMBDA`
- `LambdaAuthorizerConfig`: authorizer function ARN, `AuthorizerResultTtlInSeconds: 0`
  *(Subscription and domain checks must run on every request — do not cache the result.)*
- `XrayEnabled: true`
- `LogConfig`: ERROR level, CloudWatch role

### WAF WebACL (`full` scope only)

Skip entirely when scope is `core`. Note in output: "WAF adds a ~$5/month fixed WebACL cost
regardless of traffic. Add it when you are ready for production by re-running with --scope full."

When generating for `full` scope:
- Rate limit rule: 500 req/IP per 5-minute window, Block action
- `AWSManagedRulesCommonRuleSet`: managed rule group, None override action
- `Scope: REGIONAL`
- Attach to AppSync ARN

### SQS queues and workers (`full` scope only)

Skip entirely when scope is `core`.

When generating for `full` scope, repeat this pattern for HandoffScoringQueue,
SessionAnalysisQueue, IntegrationEventsQueue, SystemPromptGeneratorQueue:
- Queue: `VisibilityTimeout=120`, `MessageRetentionPeriod=86400` (1 day),
  `RedrivePolicy maxReceiveCount=3` → DLQ
- DLQ: `MessageRetentionPeriod=1209600` (14 days)
- CloudWatch Alarm: `ApproximateNumberOfMessagesVisible >= 1` on DLQ, `Period=60s` → SNS alert

### crawler/Dockerfile + crawler.py (`full` scope only)

Skip entirely when scope is `core`.

When generating for `full` scope, produce a complete crawler that:
- Reads jobs from `CRAWL_JOB_QUEUE_NAME` SQS queue (long-poll, 20s)
- Each job carries: `tenant_id`, `urls` (list), `crawl_config` (depth, page limit)
- Fetches each URL, extracts text content (strip HTML)
- Writes extracted content to `s3://data-bucket/documents/{tenant_id}/{filename}.txt`
- Writes metadata JSON alongside each file
- Updates DynamoDB CONFIG item: `crawl_status = COMPLETED`, `last_crawl = {iso_timestamp}`
- On failure: sets `crawl_status = FAILED`, re-raises so ECS marks the task failed

Dockerfile: `python:3.12-slim`, installs `boto3`, `requests`, `beautifulsoup4`, `lxml`.

ECS task definition in IaC: `Cpu: 512`, `Memory: 1024`, task role scoped to S3 write +
DynamoDB update + SQS consume.

### widget/ — three-file layer split

Apply the same layer discipline as the backend: **API → Services → Domain**.
Each file has one responsibility. Never let a concern bleed across files.

**Layer rules (enforce strictly):**
- `auth.js` — token fetch and refresh only. No DOM, no WebSocket, no state.
- `connection.js` — AppSync WebSocket lifecycle only. No DOM, no token logic.
- `widget.js` — thin public API coordinator. Delegates immediately to auth/connection.
  No business logic here. If `widget.js` grows beyond ~50 lines it is doing too much.
- Domain helper functions (`formatMessage`, `buildGraphQLPayload`, etc.) must be **pure**:
  no side effects, no external references, deterministic output. Keep them in `widget.js`
  or a `utils.js` if they accumulate. Pure functions are naturally testable without a framework.

**Connection state machine (mandatory — no boolean flags):**

```javascript
const State = Object.freeze({
  DISCONNECTED:  'DISCONNECTED',
  CONNECTING:    'CONNECTING',
  CONNECTED:     'CONNECTED',
  RECONNECTING:  'RECONNECTING',
});
```

State transitions must be explicit function calls. Reading state must never trigger a
transition. Boolean flags (`isConnected`, `isReconnecting`) are banned — they create
silent race conditions when two async paths run simultaneously.

**widget/auth.js** — complete implementation:
- Export async `fetchToken(tenantId, sessionTokenUrl)` → returns JWT string
- Export async `refreshToken(tenantId, sessionTokenUrl)` → same, used on 401
- Cache token in module scope; expose `getToken()` for connection layer to read
- On fetch failure: throw with a descriptive message (caller decides retry behaviour)
- No DOM access, no WebSocket references

**widget/connection.js** — complete implementation:
- Export `connect({ realtimeUrl, graphqlUrl, getToken, onMessage, onError, onStateChange })`
- Manages the AppSync WebSocket protocol (subprotocol: `graphql-ws`)
- Uses the State machine above; calls `onStateChange(newState)` on every transition
- On 401 from AppSync: calls `onError({ type: 'AUTH_EXPIRED' })` — caller refreshes token
  and calls `reconnect(newToken)`. Connection layer does NOT fetch tokens itself.
- Export `sendMessage(tenantId, sessionId, message)` — only callable in CONNECTED state;
  throws if called in any other state
- Export `disconnect()` — closes socket cleanly, transitions to DISCONNECTED

**widget/widget.js** — thin coordinator, complete implementation:
- IIFE wrapping everything; exports `window.ChatWidget`
- On load: resolve config (script tag attrs → config.json → hardcoded defaults)
- Call `auth.fetchToken(tenantId, sessionTokenUrl)`
- Call `connection.connect(...)` passing `auth.getToken` as the token supplier
- On `AUTH_EXPIRED` error from connection: call `auth.refreshToken()`, then `connection.reconnect()`
- Public API exposed on `window.ChatWidget`:
  - `sendMessage(text)` → delegates to `connection.sendMessage(...)`
  - `onMessage(callback)` → registers listener, called when AppSync pushes a reply
  - `onError(callback)` → registers error listener
  - `onStateChange(callback)` → registers connection state listener

**widget/config.json** — placed alongside widget files in S3/CloudFront:
```json
{
  "graphqlUrl":      "https://your-appsync-id.appsync-api.{region}.amazonaws.com/graphql",
  "realtimeUrl":     "wss://your-appsync-id.appsync-realtime-api.{region}.amazonaws.com/graphql",
  "sessionTokenUrl": "https://your-api-id.execute-api.{region}.amazonaws.com/session"
}
```

### widget/demo/index.html — browser test page

The widget layer owns the connection. The demo page owns the UI. Never merge them.

Generate a complete self-contained HTML page:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Chat Widget Demo</title>
  <style>
    body { font-family: sans-serif; max-width: 600px; margin: 40px auto; padding: 0 16px; }
    #messages { border: 1px solid #ddd; border-radius: 8px; height: 400px;
                overflow-y: auto; padding: 12px; margin-bottom: 12px; }
    .msg { margin: 8px 0; padding: 8px 12px; border-radius: 6px; max-width: 80%; }
    .msg.user      { background: #0070f3; color: #fff; margin-left: auto; text-align: right; }
    .msg.assistant { background: #f1f3f5; color: #111; }
    .msg.system    { color: #888; font-size: 0.85em; text-align: center; max-width: 100%; }
    #controls { display: flex; gap: 8px; }
    #input  { flex: 1; padding: 8px; border: 1px solid #ddd; border-radius: 6px; resize: none; }
    #send   { padding: 8px 16px; background: #0070f3; color: #fff;
               border: none; border-radius: 6px; cursor: pointer; }
    #send:disabled { opacity: 0.5; cursor: not-allowed; }
    #status { font-size: 0.8em; color: #888; margin-top: 6px; }
  </style>
</head>
<body>
  <h2>Chat Widget — Local Test</h2>
  <div id="messages"></div>
  <div id="controls">
    <textarea id="input" rows="2" placeholder="Type a message..." disabled></textarea>
    <button id="send" disabled>Send</button>
  </div>
  <div id="status">Connecting...</div>

  <script src="../widget.js" data-widget-id="test-tenant-id"></script>
  <script>
    const messages = document.getElementById('messages');
    const input    = document.getElementById('input');
    const send     = document.getElementById('send');
    const status   = document.getElementById('status');

    function appendMessage(role, content) {
      const div = document.createElement('div');
      div.className = `msg ${role}`;
      div.textContent = content;
      messages.appendChild(div);
      messages.scrollTop = messages.scrollHeight;
    }

    ChatWidget.onStateChange(state => {
      status.textContent = state;
      const ready = state === 'CONNECTED';
      input.disabled  = !ready;
      send.disabled   = !ready;
    });

    ChatWidget.onMessage(msg => appendMessage('assistant', msg.content));

    ChatWidget.onError(err => appendMessage('system', `Error: ${err.type}`));

    send.addEventListener('click', () => {
      const text = input.value.trim();
      if (!text) return;
      appendMessage('user', text);
      input.value = '';
      ChatWidget.sendMessage(text);
    });

    input.addEventListener('keydown', e => {
      if (e.key === 'Enter' && !e.shiftKey) { e.preventDefault(); send.click(); }
    });
  </script>
</body>
</html>
```

Test locally: `cd widget && python -m http.server 8080`, then open
`http://localhost:8080/demo/index.html` in a browser.

---

## Step 4 — Generate .env.example

Scope the env file to what was generated. For `core`, omit queue URLs and crawler vars.

```bash
# Control plane
WIDGET_JWT_SECRET=          # must match SecretsManager value — same secret, both planes
CHAT_TABLE_NAME=chat-widget-dev-chat-sessions
SUBSCRIPTION_TABLE_NAME=chat-widget-dev-subscriptions

# Execution plane
CHAT_TABLE_NAME=chat-widget-dev-chat-sessions
TOKEN_USAGE_TABLE_NAME=chat-widget-dev-token-usage
KNOWLEDGE_BASE_ID=           # from SSM after Bedrock KB provisioned
JWT_SECRET_ARN=              # SecretsManager ARN
CHAT_MODEL_ID=us.amazon.nova-2-lite-v1:0

# --- full scope only ---
HANDOFF_SCORING_QUEUE_URL=
SESSION_ANALYSIS_QUEUE_URL=
INTEGRATION_EVENTS_QUEUE_URL=
SCORER_MODEL_ID=amazon.nova-micro-v1:0
ANALYST_MODEL_ID=amazon.nova-micro-v1:0
CRAWL_JOB_QUEUE_NAME=chat-widget-dev-crawl-jobs
DATA_BUCKET=chat-widget-dev-data

AWS_REGION=us-east-1
```

---

## Step 5 — Print deploy instructions after writing all files

For `core` scope:
```
DEPLOY ORDER (core) — 4 steps:

1. Deploy the stateful layer (DynamoDB + S3 + OpenSearch Serverless + Bedrock KB + JWT secret):

   sam deploy --guided --template infra/main.yml

   SAM will prompt for stack name (e.g. {project}-{env}-infra), region, and create its own
   artifacts bucket. All resources — including the Bedrock Knowledge Base and JWT secret —
   are provisioned automatically. No console steps needed.

   ⚠️  OpenSearch Serverless collection creation takes 3–5 minutes. SAM waits for it.
   ⚠️  Note the stack name — you will pass it to the seed script in step 4.

2. Deploy the compute layer (Lambda + AppSync + HTTP API Gateway):

   sam build --template backend/templates/sam_template.yml
   sam deploy --guided --template backend/templates/sam_template.yml

   SAM will prompt for stack name (e.g. {project}-{env}-compute). When asked for parameter
   values, the compute stack reads JWT secret ARN and KB ID from SSM automatically — no
   manual copy-paste of ARNs needed.

3. Populate widget/config.json from stack outputs:

   chmod +x scripts/update-config.sh
   ./scripts/update-config.sh {project}-{env}-compute

   This writes the three AppSync/SessionToken URLs into widget/config.json in one command.

4. Seed test data and verify locally:

   chmod +x scripts/seed-tenant.sh
   ./scripts/seed-tenant.sh {project}-{env}-infra

   Then start the local server:
   cd widget && python -m http.server 8080

   Open http://localhost:8080/demo/index.html — the widget should reach CONNECTED state
   and respond to messages. The input textarea enables automatically once the WebSocket
   handshake completes.

When ready to add WAF, async workers, and the web crawler:
  re-run /build-chat-widget --scope full
```

For `full` scope, add after step 2:
```
2b. Build and push crawler Docker image to ECR:
    aws ecr get-login-password | docker login --username AWS --password-stdin \
      {account}.dkr.ecr.{region}.amazonaws.com
    docker build -t {project}-crawler ./crawler
    docker tag {project}-crawler:latest \
      {account}.dkr.ecr.{region}.amazonaws.com/{project}-crawler:latest
    docker push {account}.dkr.ecr.{region}.amazonaws.com/{project}-crawler:latest

Note: WAF adds a fixed ~$5/month WebACL cost from day one regardless of traffic volume.
```

---

## Step 6 — Generate README.md at project root

Write `{project_name}/README.md` as the final file after all code is written.

The README is what the user reads cold — it must be fully self-contained. Use real values
from the build (actual project name, scope chosen, IaC stack, region, concrete stack name
suggestions). Never leave generic placeholders where the real value is already known.

**Required content — Claude decides the prose and structure:**

- What was built: the actual directory tree for the scope generated, with a one-line
  explanation of each file's role
- Prerequisites: tooling versions needed, how to verify credentials are active
- Deploy order: exact commands for the IaC stack that was generated, in the right sequence,
  with the OpenSearch Serverless timing warning for the infra stack. Include the `full` scope
  Docker/ECR step only if `full` was generated.
- Scripts: what `update-config.sh` and `seed-tenant.sh` do, their arguments, when to run them
- Local test: how to serve the demo page and what a successful connection looks like
- Architecture summary: the two-plane pattern in plain language, and the JWT contract table
  (iss, aud, exp, algorithm, secret source) — this is the most critical thing to get wrong,
  so it must be in the README
- Shared Knowledge Base: one KB per deployment, all tenants use the same KB ID, where to
  upload documents, how to trigger a Bedrock ingestion job
- What's next: if `core` was built, explain how to upgrade to `full` scope

---

## Architecture constraints — never deviate from these

- **Two-plane design is mandatory.** `SessionTokenFunction` (HTTP API Gateway) is the control
  plane — it issues JWTs after validating origin and subscription. AppSync + Lambda is the
  execution plane — it validates JWTs. The planes never call each other; DynamoDB CONFIG is
  the only shared interface.
- **Backend follows entry point → services → domain layer direction.** `handler.py` is ~30
  lines max. Services orchestrate. Domain functions are pure. Adapters are the only boto3
  boundary. Never reverse the dependency direction.
- **Read paths never write.** `get_tenant_config`, `load_history`, `check_quota` are
  read-only. Any mutation is a separate explicit write call in the service layer.
- **AWS clients and secrets are initialised at module scope**, not inside `lambda_handler`.
- **`SessionTokenFunction` must validate origin AND subscription before issuing a token.**
  Do not short-circuit either check. A token issued without a subscription check allows
  unsubscribed tenants to reach Bedrock.
- **JWT validation happens at two independent points:** control plane at token issuance
  (Origin + subscription check) and Lambda Authorizer at every AppSync request.
  Do not collapse these into one.
- **Bedrock is invoked synchronously in ChatApiFunction.** The visitor is waiting.
  Do not queue the main AI response.
- **DynamoDB partition key is always `TENANT#{tenant_id}`.** Never use bare `tenant_id` as PK.
- **No secrets or credentials in Lambda environment variables.** JWT secret goes in
  SecretsManager. Runtime config goes in SSM Parameter Store.
- **`AuthorizerResultTtlInSeconds` must be `0`.** Subscription and domain checks must run
  live on every request — do not cache the authorizer result.
- **SQS publishes in ChatApiFunction must be conditional** on queue URL env vars being set,
  so `core` scope works cleanly without any queues configured.
- **Every SQS queue (`full` scope) must have a DLQ and a CloudWatch alarm.** No exceptions.
- **WAF (`full` scope) rate limit is 500 req/IP per 5 minutes.** Do not lower this.
- **Async workers use `amazon.nova-micro-v1:0`.** Never use Nova 2 Lite for classification
  or summarisation tasks — it is ~6x more expensive with no quality gain.
- **Messages are stored as a list on the session item**, not as individual DynamoDB items.
  One item per session. Append messages atomically.
- **AppSync URL is hot-swappable via `config.json`** placed alongside `widget.js` in S3.
  Never hardcode the AppSync endpoint in the widget bundle.
- **Widget follows API → Services → Domain layer direction.** `auth.js`, `connection.js`,
  and `widget.js` are separate files with single responsibilities. No concern bleeds across
  files. `widget.js` must not grow beyond ~50 lines.
- **Connection state is an explicit enum, never boolean flags.** `DISCONNECTED | CONNECTING |
  CONNECTED | RECONNECTING`. Reading state never triggers a transition.
- **Domain helper functions are pure.** No side effects, no external references. If a function
  cannot be called twice with the same arguments and return the same result, it is not pure
  and does not belong in the domain layer.
- **The demo page owns the UI; the widget owns the connection.** Never inject DOM from
  `widget.js`. The public API is `sendMessage`, `onMessage`, `onError`, `onStateChange`.
