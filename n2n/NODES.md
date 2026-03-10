# NODES.md — N2N Toolz Built-In Node Reference

> All nodes follow the [NODE_SCHEMA.md](./NODE_SCHEMA.md) contract.
> Every node has: `type`, `inputs`, `outputs`, `configSchema`, `execute()`, `generateCode()`

---

## Category: 🔴 Trigger Nodes

### HTTP Trigger

| Field | Value |
|---|---|
| type | `http-trigger` |
| Inputs | None |
| Outputs | `request` (object), `headers` (object), `params` (object) |

**Config:**
```json
{
  "method": "GET | POST | PUT | DELETE | PATCH",
  "path": "/api/endpoint",
  "bodyParser": "json | urlencoded | raw"
}
```
**Generated Code:**
```javascript
router.post('/api/endpoint', async (req, res, next) => { ... });
```

---

### Cron Trigger

| Field | Value |
|---|---|
| type | `cron-trigger` |
| Inputs | None |
| Outputs | `tick` (object: `{ timestamp, cronExpression }`) |

**Config:**
```json
{ "expression": "0 * * * *", "timezone": "Asia/Kolkata" }
```
**Generated Code:**
```javascript
import cron from 'node-cron';
cron.schedule('0 * * * *', async () => { ... }, { timezone: 'Asia/Kolkata' });
```

---

### Manual Trigger

| Field | Value |
|---|---|
| type | `manual-trigger` |
| Inputs | `payload` (any — injected at runtime) |
| Outputs | `data` (any) |

Used for direct invocation. Generates a named exported async function.

---

### Event Trigger

| Field | Value |
|---|---|
| type | `event-trigger` |
| Inputs | None |
| Outputs | `event` (object), `data` (any) |

**Config:**
```json
{ "emitter": "globalEvents", "eventName": "user.created" }
```

---

### WebSocket Trigger

| Field | Value |
|---|---|
| type | `websocket-trigger` |
| Inputs | None |
| Outputs | `message` (string/object), `socket` (object) |

**Config:**
```json
{ "path": "/ws", "parseJSON": true }
```

---

## Category: 🔀 Flow Control

### If / Else

| Field | Value |
|---|---|
| type | `if-else` |
| Inputs | `data` (any) |
| Outputs | `true` (any), `false` (any) |

**Config:**
```json
{ "condition": "input.status === 'active'" }
```
**Generated Code:**
```javascript
if (node_001_out.status === 'active') { ... } else { ... }
```

---

### Switch

| Field | Value |
|---|---|
| type | `switch` |
| Inputs | `value` (any) |
| Outputs | Dynamic: one port per case + `default` |

**Config:**
```json
{
  "expression": "input.role",
  "cases": ["admin", "user", "guest"]
}
```

---

### Loop (forEach)

| Field | Value |
|---|---|
| type | `loop-foreach` |
| Inputs | `collection` (array) |
| Outputs | `item` (any), `index` (number), `done` (array) |

**Generated Code:**
```javascript
const results = [];
for (const [index, item] of collection.entries()) {
  // inner nodes execute here
  results.push(loopResult);
}
```

---

### While Loop

| Field | Value |
|---|---|
| type | `loop-while` |
| Inputs | `initial` (any) |
| Outputs | `iteration` (any), `result` (any) |

**Config:**
```json
{ "condition": "state.count < 10", "maxIterations": 1000 }
```

---

### Try / Catch

| Field | Value |
|---|---|
| type | `try-catch` |
| Inputs | `data` (any) |
| Outputs | `success` (any), `error` (Error object) |

**Generated Code:**
```javascript
try {
  // try-branch nodes
} catch (err) {
  // catch-branch nodes
}
```

---

### Merge

| Field | Value |
|---|---|
| type | `merge` |
| Inputs | `branch1`, `branch2`, ... `branchN` (dynamic) |
| Outputs | `merged` (array) |

**Config:**
```json
{ "strategy": "all | race | allSettled", "inputCount": 3 }
```

---

### Split

| Field | Value |
|---|---|
| type | `split` |
| Inputs | `data` (any) |
| Outputs | `out1`, `out2`, ... `outN` (same data, fan-out) |

**Config:**
```json
{ "outputCount": 3 }
```

---

### Delay

| Field | Value |
|---|---|
| type | `delay` |
| Inputs | `data` (any) |
| Outputs | `data` (any — after delay) |

**Config:**
```json
{ "duration": 1000, "unit": "ms | s | m" }
```

---

## Category: 🔄 Data Transform

### Map

| Field | Value |
|---|---|
| type | `data-map` |
| Inputs | `collection` (array) |
| Outputs | `result` (array) |

**Config:**
```json
{ "expression": "item => ({ id: item.id, name: item.name.toUpperCase() })" }
```

---

### Filter

| Field | Value |
|---|---|
| type | `data-filter` |
| Inputs | `collection` (array) |
| Outputs | `result` (array), `rejected` (array) |

**Config:**
```json
{ "expression": "item => item.active === true" }
```

---

### Reduce

| Field | Value |
|---|---|
| type | `data-reduce` |
| Inputs | `collection` (array) |
| Outputs | `result` (any) |

**Config:**
```json
{ "expression": "(acc, item) => acc + item.value", "initialValue": "0" }
```

---

### JSON Parse

| Field | Value |
|---|---|
| type | `json-parse` |
| Inputs | `raw` (string) |
| Outputs | `parsed` (object/array), `error` (error) |

---

### JSON Stringify

| Field | Value |
|---|---|
| type | `json-stringify` |
| Inputs | `data` (any) |
| Outputs | `json` (string) |

**Config:**
```json
{ "indent": 2 }
```

---

### Set Variable

| Field | Value |
|---|---|
| type | `set-variable` |
| Inputs | `data` (any) |
| Outputs | `data` (any — passthrough) |

**Config:**
```json
{ "name": "myVar", "value": "expression or static", "scope": "workflow | global" }
```

---

### Template (Handlebars)

| Field | Value |
|---|---|
| type | `template` |
| Inputs | `data` (object) |
| Outputs | `result` (string) |

**Config:**
```json
{ "template": "Hello, {{name}}! Your order {{orderId}} is ready." }
```

---

## Category: 📤 Output Nodes

### HTTP Response

| Field | Value |
|---|---|
| type | `http-response` |
| Inputs | `body` (any), `headers` (object), `statusCode` (number) |
| Outputs | None (terminal) |

**Config:**
```json
{ "defaultStatus": 200, "contentType": "application/json" }
```

---

### Console Log

| Field | Value |
|---|---|
| type | `console-log` |
| Inputs | `data` (any) |
| Outputs | `data` (any — passthrough) |

**Config:**
```json
{ "level": "info | warn | error | debug", "label": "prefix label" }
```

---

### Return Value

| Field | Value |
|---|---|
| type | `return-value` |
| Inputs | `value` (any) |
| Outputs | None (terminal) |

Wraps execution result as function return value in generated code.

---

### Throw Error

| Field | Value |
|---|---|
| type | `throw-error` |
| Inputs | `message` (string), `data` (any) |
| Outputs | None (terminal) |

**Config:**
```json
{ "errorType": "Error | ValidationError | NotFoundError", "statusCode": 400 }
```

---

## Category: 🌐 HTTP / API

### HTTP Request

| Field | Value |
|---|---|
| type | `http-request` |
| Inputs | `body` (any), `headers` (object) |
| Outputs | `response` (object), `status` (number), `error` (error) |

**Config:**
```json
{
  "url": "https://api.example.com/data",
  "method": "GET | POST | PUT | DELETE | PATCH",
  "headers": {},
  "timeout": 10000,
  "auth": "none | bearer | basic | apiKey"
}
```

---

### GraphQL Client

| Field | Value |
|---|---|
| type | `graphql-client` |
| Inputs | `variables` (object) |
| Outputs | `data` (object), `errors` (array) |

**Config:**
```json
{
  "endpoint": "https://api.example.com/graphql",
  "query": "query GetUser($id: ID!) { user(id: $id) { name email } }",
  "headers": {}
}
```

---

### OAuth2 Handler

| Field | Value |
|---|---|
| type | `oauth2-handler` |
| Inputs | `code` (string) |
| Outputs | `accessToken` (string), `refreshToken` (string), `expiry` (number) |

**Config:**
```json
{
  "clientId": "",
  "clientSecret": "",
  "tokenUrl": "",
  "redirectUri": ""
}
```

---

## Category: 🗄️ Database

### MySQL Query

| Field | Value |
|---|---|
| type | `mysql-query` |
| Inputs | `params` (array) |
| Outputs | `rows` (array), `affectedRows` (number), `error` (error) |

**Config:**
```json
{
  "connection": "env:MYSQL_URL",
  "query": "SELECT * FROM users WHERE id = ?",
  "paramSource": "input | config"
}
```

---

### PostgreSQL Query

| type | `postgres-query` |
|---|---|
| Inputs | `params` (array) |
| Outputs | `rows` (array), `rowCount` (number), `error` (error) |

**Config:**
```json
{ "connection": "env:PG_URL", "query": "SELECT $1::text AS result" }
```

---

### MongoDB Query

| type | `mongodb-query` |
|---|---|
| Inputs | `filter` (object), `update` (object) |
| Outputs | `result` (any), `error` (error) |

**Config:**
```json
{
  "connection": "env:MONGO_URI",
  "collection": "users",
  "operation": "find | findOne | insertOne | updateOne | deleteOne | aggregate"
}
```

---

### Redis Get / Set

| type | `redis-get-set` |
|---|---|
| Inputs | `key` (string), `value` (any) |
| Outputs | `result` (any), `error` (error) |

**Config:**
```json
{ "connection": "env:REDIS_URL", "operation": "get | set | del | hget | hset", "ttl": 3600 }
```

---

## Category: 📁 File System

### Read File

| type | `fs-read` |
|---|---|
| Inputs | `filePath` (string) |
| Outputs | `content` (string/Buffer), `error` (error) |

**Config:**
```json
{ "encoding": "utf8 | base64 | binary | buffer", "staticPath": "" }
```

---

### Write File

| type | `fs-write` |
|---|---|
| Inputs | `content` (string/Buffer), `filePath` (string) |
| Outputs | `success` (boolean), `error` (error) |

**Config:**
```json
{ "encoding": "utf8", "append": false, "createDirs": true }
```

---

### Watch File / Directory

| type | `fs-watch` |
|---|---|
| Inputs | None (trigger) |
| Outputs | `event` (string), `filename` (string) |

**Config:**
```json
{ "path": "./data", "events": ["change", "rename"], "recursive": true }
```

---

## Category: 🔐 Auth & Security

### JWT Sign

| type | `jwt-sign` |
|---|---|
| Inputs | `payload` (object) |
| Outputs | `token` (string) |

**Config:**
```json
{ "secret": "env:JWT_SECRET", "expiresIn": "1h", "algorithm": "HS256" }
```

---

### JWT Verify

| type | `jwt-verify` |
|---|---|
| Inputs | `token` (string) |
| Outputs | `decoded` (object), `error` (error) |

**Config:**
```json
{ "secret": "env:JWT_SECRET", "algorithms": ["HS256"] }
```

---

### Hash (bcrypt)

| type | `hash-bcrypt` |
|---|---|
| Inputs | `value` (string) |
| Outputs | `hash` (string) |

**Config:**
```json
{ "saltRounds": 10, "operation": "hash | compare" }
```

---

### Encrypt / Decrypt (AES)

| type | `aes-cipher` |
|---|---|
| Inputs | `data` (string) |
| Outputs | `result` (string) |

**Config:**
```json
{ "key": "env:AES_KEY", "iv": "env:AES_IV", "operation": "encrypt | decrypt", "algorithm": "aes-256-cbc" }
```

---

## Category: 📨 Messaging

### SMTP Email

| type | `smtp-email` |
|---|---|
| Inputs | `to` (string), `subject` (string), `body` (string) |
| Outputs | `messageId` (string), `error` (error) |

**Config:**
```json
{
  "host": "env:SMTP_HOST",
  "port": 587,
  "secure": false,
  "user": "env:SMTP_USER",
  "pass": "env:SMTP_PASS",
  "from": "noreply@example.com"
}
```

---

### Slack Message

| type | `slack-message` |
|---|---|
| Inputs | `text` (string), `attachments` (array) |
| Outputs | `ts` (string), `error` (error) |

**Config:**
```json
{ "webhookUrl": "env:SLACK_WEBHOOK", "channel": "#general", "username": "N2N Bot" }
```

---

### Kafka Producer / Consumer

| type | `kafka-producer` / `kafka-consumer` |
|---|---|
| Inputs | `message` (any) |
| Outputs | `offset` (number), `error` (error) |

**Config:**
```json
{ "brokers": ["env:KAFKA_BROKER"], "topic": "my-topic", "groupId": "my-group" }
```

---

## Category: ⚙️ Custom Code

### Function Node

| type | `function-node` |
|---|---|
| Inputs | `data` (any) |
| Outputs | `result` (any), `error` (error) |

**Config:**
```json
{ "code": "const result = input.data.map(x => x * 2);\nreturn { result };" }
```

The code runs inside a sandboxed async function. `input`, `env`, and `logger` are injected.

---

### Shell Command

| type | `shell-command` |
|---|---|
| Inputs | `args` (string[]) |
| Outputs | `stdout` (string), `stderr` (string), `exitCode` (number) |

**Config:**
```json
{ "command": "ls -la", "cwd": "./", "shell": true, "timeout": 10000 }
```

> ⚠️ Disabled in sandbox mode. Only available in dev and export modes.