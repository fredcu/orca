# Orca Fork — Structured Agent Message Bus 设计文档

日期：2026-07-08

## 1. 背景

本设计文档面向 `fredcu/orca` fork。目标是在不引入 ChatGPT bridge、不重写 Orca 主架构、不破坏 upstream Orca orchestration 语义的前提下，将 ForgeAgentDesktop 中较成熟的 structured message bus 思路融合进 Orca。

当前 Orca 已经具备以下基础能力：

- Electron + React + TypeScript 桌面架构。
- 支持 Codex、Claude Code、OpenCode、Cursor、Grok、Copilot CLI、Goose 等 CLI agents。
- 支持 terminal splits、parallel worktrees、agent launch、mobile companion、SSH worktrees。
- 已存在 experimental orchestration 层，包括 messages、tasks、dispatch contexts、decision gates、coordinator runs。

ForgeAgentDesktop 的 message bus 强项在于 structured envelope：消息不只是普通文本，而是包含 `protocolVersion`、`summary`、`requestText`、`acceptanceCriteria`、`risks`、`artifactRefs`、`ackState`、`contextManifestHash`、`policyHash`、`envelopeHash`、`inReplyTo` 等字段。

因此，本设计的核心不是从零实现一套独立 bus，而是在 Orca 现有 orchestration DB/runtime/CLI 上增加一个结构化消息协议层。

---

## 2. 设计目标

### 2.1 核心目标

1. 在 Orca fork 中增加 structured agent-to-agent message bus。
2. 复用 Orca 现有 runtime、terminal handle、orchestration DB、CLI 和 agent terminal 管理能力。
3. 引入 Forge 风格的 structured envelope。
4. 支持 agent 间结构化通信：
   - send
   - inbox/list
   - show
   - reply
   - ack
   - reject
   - thread 查询
   - compatibility check
5. 支持消息携带：
   - summary
   - requestText
   - acceptanceCriteria
   - risks/open questions
   - artifactRefs
   - contextManifestHash
   - policyHash
   - envelopeHash
   - inReplyTo
6. 明确区分：
   - message bus：结构化通信协议
   - orchestration：有 coordinator 监督的 task lifecycle
   - handoff：ownership transfer
   - dispatch：supervised assignment

### 2.2 非目标

第一阶段不做以下内容：

1. 不考虑 ChatGPT bridge。
2. 不做 ChatGPT webview / DOM injection / reply harvesting。
3. 不重写 Orca UI。
4. 不重写 Orca PTY / terminal 热路径。
5. 不替换 Orca orchestration。
6. 不把 ForgeAgentDesktop 的 Tauri IPC 直接迁移进 Orca。
7. 不默认自动注入 bus message 到 agent TUI。
8. 不默认创建 task/dispatch。

---

## 3. 当前 Orca 能力基线

### 3.1 已有 orchestration 文件

当前 upstream Orca 已经包含以下相关文件：

```text
src/main/runtime/orchestration/db.ts
src/main/runtime/orchestration/types.ts
src/main/runtime/orchestration/formatter.ts
src/main/runtime/orchestration/preamble.ts
src/main/runtime/rpc/methods/orchestration.ts
src/cli/handlers/orchestration.ts
src/cli/specs/orchestration.ts
skills/orchestration/SKILL.md
```

### 3.2 已有 orchestration DB

Orca orchestration DB 已经有：

```text
messages
tasks
dispatch_contexts
decision_gates
coordinator_runs
```

其中 `messages` 已有字段：

```text
id
from_handle
to_handle
subject
body
type
priority
thread_id
payload
read
sequence
created_at
delivered_at
```

这些字段足够承载第一阶段的 structured message bus。

### 3.3 已有 message type

当前 Orca orchestration message type 包括：

```text
status
dispatch
worker_done
merge_ready
escalation
handoff
decision_gate
heartbeat
```

这组类型偏 orchestration lifecycle。Structured message bus 会增加更细的协议语义，但第一阶段不建议马上扩展 DB 的 CHECK constraint，而是把细粒度类型放在 `payload.messageType` 中。

### 3.4 已有 read / delivered 语义

Orca 已有：

```text
read
delivered_at
getUnreadMessages
getUndeliveredUnreadMessages
markAsRead
markAsDelivered
getThreadMessagesFor
```

这些能力应当继续复用。不要把 `read`、`delivered_at` 和新的 `ackState` 混为一谈。

---

## 4. 总体架构

### 4.1 架构原则

采用“增强 Orca orchestration，而不是替换 Orca orchestration”的方式。

```text
CLI / Renderer / Agent terminal
        ↓
Agent Message Bus API
        ↓
Structured Envelope Adapter
        ↓
Orca Orchestration DB / Runtime
        ↓
Terminal Handle / Worktree / PTY Runtime
```

### 4.2 建议目录结构

新增：

```text
src/shared/agent-message-bus/types.ts
src/shared/agent-message-bus/envelope.ts
src/shared/agent-message-bus/validation.ts
src/shared/agent-message-bus/hash.ts
src/main/runtime/orchestration/agent-message-adapter.ts
src/main/runtime/orchestration/agent-message-service.ts
src/main/runtime/orchestration/agent-message-compatibility.ts
src/main/runtime/orchestration/agent-message-formatter.ts
src/cli/specs/bus.ts
src/cli/handlers/bus.ts
```

后续 UI 阶段再增加：

```text
src/renderer/src/components/message-bus/
src/renderer/src/store/slices/messageBus.ts
```

### 4.3 为什么第一阶段不新建独立 DB

Orca 已经有 orchestration DB。如果单独建 message bus DB，会导致：

1. terminal handle 映射重复。
2. read/delivered 状态重复。
3. CLI 查询重复。
4. message bus 和 orchestration 状态割裂。
5. 后续 UI 需要同时读取两个来源。
6. upstream merge 冲突风险增加。

因此第一阶段复用 `messages.payload` 保存 structured envelope。

---

## 5. 核心概念边界

### 5.1 Message Bus

Message bus 是 agent-to-agent 的结构化通信协议。

它负责：

```text
send
reply
ack
reject
thread
artifact references
acceptance criteria
risk/open question tracking
compatibility check
```

它不负责：

```text
创建 worktree
启动 agent
监督 task 完成
自动 merge
自动 monitor worker
自动执行 artifact
```

### 5.2 Orchestration

Orchestration 是 Orca 已有的 supervised coordination lifecycle。

它负责：

```text
task DAG
dispatch
worker_done
heartbeat
decision_gate
coordinator wait loop
```

### 5.3 Handoff

Handoff 是 ownership transfer。

规则：

```text
handoff 表示把上下文、任务或责任移交给另一个 agent。
handoff 本身不意味着原 agent 要继续 monitor。
handoff 不默认创建 task / dispatch。
handoff 不默认等待 worker_done。
```

### 5.4 Dispatch

Dispatch 是 supervised assignment。

规则：

```text
dispatch 只在 coordinator 明确监督 worker 时使用。
dispatch 应当绑定 taskId / dispatchId。
worker_done 只对 dispatch lifecycle 有完成权威性。
```

### 5.5 Review Request

`review_request` 可以有两种模式：

```text
one-shot review request:
  只是给另一个 agent 发结构化审查请求，不创建 task/dispatch。

supervised review dispatch:
  由 orchestration task/dispatch 包装，worker_done 才有 lifecycle 完成意义。
```

第一阶段只实现 one-shot review request。

---

## 6. Structured Envelope 设计

### 6.1 TypeScript 类型

新增 `src/shared/agent-message-bus/types.ts`：

```ts
export type AgentMessageType =
  | 'status'
  | 'question'
  | 'answer'
  | 'handoff'
  | 'review_request'
  | 'review_response'
  | 'decision'
  | 'artifact'
  | 'blocked'
  | 'done'

export type AgentMessageAckState =
  | 'pending'
  | 'acknowledged'
  | 'rejected'

export type AgentMessagePriority =
  | 'normal'
  | 'high'
  | 'urgent'

export type AgentMessageEnvelopeV1 = {
  kind: 'agent_message_envelope_v1'
  protocolVersion: 'agent.message.v1'

  projectId: string | null
  threadId: string
  round: number

  fromSessionId: string
  toSessionId: string

  messageType: AgentMessageType
  priority: AgentMessagePriority

  summary: string | null
  requestText: string | null
  acceptanceCriteria: string | null

  risks: string[]
  artifactRefs: string[]

  contextManifestHash: string | null
  policyHash: string | null
  envelopeHash: string | null

  ackState: AgentMessageAckState
  ackNote?: string | null
  ackedAt?: string | null

  inReplyTo: string | null
  createdAt: string
}
```

### 6.2 字段说明

| 字段 | 说明 |
|---|---|
| `kind` | envelope discriminator |
| `protocolVersion` | bus 协议版本 |
| `projectId` | repo/worktree/project scope，可为空 |
| `threadId` | thread id |
| `round` | 协作轮次 |
| `fromSessionId` | sender terminal handle |
| `toSessionId` | receiver terminal handle |
| `messageType` | structured message type |
| `priority` | normal/high/urgent |
| `summary` | 简短摘要 |
| `requestText` | 完整请求正文 |
| `acceptanceCriteria` | 验收标准 |
| `risks` | 风险和开放问题 |
| `artifactRefs` | 文件、diff、terminal、task、PR 等引用 |
| `contextManifestHash` | 上下文 manifest hash |
| `policyHash` | policy/rules hash |
| `envelopeHash` | envelope 内容 hash |
| `ackState` | pending/acknowledged/rejected |
| `ackNote` | ack/reject 附加说明 |
| `ackedAt` | ack/reject 时间 |
| `inReplyTo` | 被回复 message id |
| `createdAt` | ISO timestamp |

---

## 7. Orca MessageRow 映射

### 7.1 第一阶段映射

第一阶段不改 DB schema，使用 `messages.payload` 保存完整 envelope。

| AgentMessageEnvelopeV1 | Orca messages |
|---|---|
| `fromSessionId` | `from_handle` |
| `toSessionId` | `to_handle` |
| `summary` | `subject` |
| `requestText` | `body` |
| `messageType` | `payload.messageType` |
| compatible coarse type | `type` |
| `priority` | `priority` |
| `threadId` | `thread_id` |
| full envelope | `payload` |
| `ackState` | `payload.ackState` |
| `artifactRefs` | `payload.artifactRefs` |
| `inReplyTo` | `payload.inReplyTo` |

### 7.2 coarse type 映射

因为 Orca `messages.type` 当前有 CHECK constraint，第一阶段用 coarse type 兼容现有值：

| Bus `messageType` | Orca `type` |
|---|---|
| `status` | `status` |
| `question` | `decision_gate` 或 `status` |
| `answer` | `status` |
| `handoff` | `handoff` |
| `review_request` | `handoff` 或 `status` |
| `review_response` | `status` |
| `decision` | `decision_gate` |
| `artifact` | `status` |
| `blocked` | `escalation` |
| `done` | `status`，除非处于 supervised dispatch |

建议默认：

```text
review_request -> status
review_response -> status
question -> status
answer -> status
blocked -> escalation
handoff -> handoff
```

原因是第一阶段不要让 `review_request` 被误解成 full handoff 或 supervised dispatch。

### 7.3 Adapter 接口

新增 `src/main/runtime/orchestration/agent-message-adapter.ts`：

```ts
export type AgentMessageInsertInput = {
  from: string
  to: string
  subject: string
  body?: string
  type?: MessageType
  priority?: MessagePriority
  threadId?: string
  payload?: string
}

export type AgentMessageProjection = {
  id: string
  fromHandle: string
  toHandle: string
  subject: string
  body: string
  threadId: string | null
  sequence: number
  read: boolean
  deliveredAt: string | null
  createdAt: string
  envelope: AgentMessageEnvelopeV1 | null
  isStructured: boolean
}

export function envelopeToMessageInsertInput(
  envelope: AgentMessageEnvelopeV1
): AgentMessageInsertInput

export function messageRowToAgentMessageProjection(
  row: MessageRow
): AgentMessageProjection
```

---

## 8. ACK / Reject 设计

### 8.1 read / delivered / ack 的区别

```text
read:
  receiver 已经读取或消费消息。

delivered:
  Orca 已经把消息展示或注入给 receiver。

acknowledged:
  receiver 明确接受消息内容、请求或责任。

rejected:
  receiver 明确拒绝消息内容、请求或责任，并可附 note。
```

不要把 `read` 当成 `acknowledged`。

### 8.2 第一阶段实现

第一阶段把 ack 写回 payload：

```json
{
  "kind": "agent_message_envelope_v1",
  "ackState": "acknowledged",
  "ackNote": "Accepted",
  "ackedAt": "2026-07-08T00:00:00.000Z"
}
```

优点：

```text
不改 schema
实现快
风险低
upstream merge 冲突小
```

缺点：

```text
group message / multi-recipient ack 不优雅
不能高效查询 ack 状态
```

### 8.3 第二阶段 ACK 表

第二阶段新增表：

```sql
CREATE TABLE IF NOT EXISTS message_acks (
  id TEXT PRIMARY KEY,
  message_id TEXT NOT NULL,
  by_handle TEXT NOT NULL,
  ack_state TEXT NOT NULL CHECK(ack_state IN ('acknowledged', 'rejected')),
  note TEXT,
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX IF NOT EXISTS idx_message_acks_message_id
  ON message_acks(message_id);

CREATE INDEX IF NOT EXISTS idx_message_acks_by_handle
  ON message_acks(by_handle);
```

---

## 9. Envelope Hash 设计

### 9.1 目的

`envelopeHash` 用于：

1. 检测 message 内容是否被修改。
2. 支持 audit/provenance。
3. 支持 reply 时引用原始请求。
4. 后续支持 artifact provenance。

### 9.2 hash 输入

不包含易变字段：

```text
createdAt
ackState
ackNote
ackedAt
read/delivered 状态
```

包含稳定字段：

```text
protocolVersion
projectId
threadId
round
fromSessionId
toSessionId
messageType
priority
summary
requestText
acceptanceCriteria
risks
artifactRefs
contextManifestHash
policyHash
inReplyTo
```

### 9.3 hash 实现

新增 `src/shared/agent-message-bus/hash.ts`：

```ts
export function canonicalizeEnvelopeForHash(
  envelope: AgentMessageEnvelopeV1
): string

export function computeEnvelopeHash(
  envelope: AgentMessageEnvelopeV1
): string
```

算法：

```text
sha256(canonicalJson)
```

canonical JSON 要求：

```text
object keys stable sort
arrays preserve order
null preserved
undefined omitted
```

---

## 10. Compatibility Check 设计

### 10.1 目标

保留 Forge message bus 中 `checkHandoffCompatibility` 的概念，但基于 Orca terminal/worktree runtime 实现。

### 10.2 检查维度

```text
from terminal exists
to terminal exists
to terminal is alive
to terminal is agent or shell
from/to 是否同 worktree
from/to 是否同 repo
to terminal 是否 idle
to terminal 是否支持 structured instruction
是否 remote / SSH
artifact path 是否对 receiver 可见
```

### 10.3 返回类型

```ts
export type AgentMessageCompatibilityReport = {
  compatible: boolean
  fromState: string | null
  toState: string | null
  blockedReason: string | null
  warnings: string[]
}
```

### 10.4 第一阶段规则

第一阶段保持保守：

```text
receiver handle 不存在 -> incompatible
receiver terminal exited -> incompatible
receiver 是 bare shell -> compatible with warning
不同 worktree -> compatible with warning
不同 repo -> compatible with warning
artifactRefs 包含不可验证路径 -> compatible with warning
```

不要因为 warning 阻止发送，除非 CLI 增加 `--strict`。

---

## 11. CLI 设计

### 11.1 新增命令组

新增独立命令组：

```bash
orca bus send
orca bus inbox
orca bus show
orca bus reply
orca bus ack
orca bus reject
orca bus check-compat
```

不建议第一阶段直接修改 `orca orchestration send`。

原因：

```text
orchestration send 是 lifecycle message。
bus send 是 structured communication。
二者语义不同，分开更安全。
```

### 11.2 send

示例：

```bash
orca bus send \
  --to codex-reviewer \
  --type review_request \
  --summary "Review message bus design" \
  --request-file docs/2026-07-08-agent-message-bus-design.md \
  --acceptance "Find correctness, lifecycle, and maintainability risks" \
  --artifact path:docs/2026-07-08-agent-message-bus-design.md \
  --risk "May confuse handoff with dispatch" \
  --json
```

参数：

```text
--from <handle>
--to <handle>
--type <messageType>
--summary <text>
--request <text>
--request-file <path>
--acceptance <text>
--risk <text> repeated
--artifact <ref> repeated
--thread-id <id>
--in-reply-to <msgId>
--priority normal|high|urgent
--strict
--json
```

### 11.3 inbox

```bash
orca bus inbox --terminal <handle> --unread --json
orca bus inbox --terminal active --limit 20
```

参数：

```text
--terminal <handle|active>
--unread
--structured-only
--type <messageType>
--limit <n>
--json
```

### 11.4 show

```bash
orca bus show --id <messageId> --json
```

### 11.5 reply

```bash
orca bus reply \
  --id <messageId> \
  --summary "Review completed" \
  --request "Findings attached" \
  --artifact path:docs/review.md \
  --json
```

Reply 规则：

```text
默认继承原 message 的 threadId。
from/to 自动反转。
inReplyTo 指向原 message id。
round = original.round + 1。
```

### 11.6 ack / reject

```bash
orca bus ack --id <messageId> --note "Accepted" --json
orca bus reject --id <messageId> --note "Receiver lacks required context" --json
```

### 11.7 check-compat

```bash
orca bus check-compat --from <handle> --to <handle> --json
```

---

## 12. Runtime Service 设计

新增 `src/main/runtime/orchestration/agent-message-service.ts`：

```ts
export class AgentMessageService {
  send(input: AgentMessageSendInput): AgentMessageProjection
  inbox(input: AgentMessageInboxInput): AgentMessageProjection[]
  show(id: string): AgentMessageProjection | null
  reply(input: AgentMessageReplyInput): AgentMessageProjection
  ack(input: AgentMessageAckInput): AgentMessageProjection
  reject(input: AgentMessageRejectInput): AgentMessageProjection
  checkCompatibility(
    input: AgentMessageCompatibilityInput
  ): AgentMessageCompatibilityReport
}
```

### 12.1 Send flow

```text
CLI/RPC input
  -> validate args
  -> resolve from handle if omitted
  -> check compatibility
  -> build envelope
  -> compute envelopeHash
  -> adapter maps envelope to OrchestrationDb.insertMessage
  -> return projection
```

### 12.2 Reply flow

```text
load original message
  -> parse original envelope if structured
  -> reverse from/to
  -> preserve threadId
  -> set inReplyTo
  -> round + 1
  -> insert message
```

### 12.3 Ack/reject flow

```text
load message
  -> parse payload
  -> if structured: update ackState/ackNote/ackedAt
  -> write updated payload back
  -> return projection
```

第一阶段如果 `OrchestrationDb` 没有 update message payload 的方法，需要增加最小方法：

```ts
updateMessagePayload(id: string, payload: string): MessageRow | undefined
```

---

## 13. Formatter / Injection 设计

### 13.1 Human-readable formatter

新增 `src/main/runtime/orchestration/agent-message-formatter.ts`。

格式示例：

```text
--- Agent Message Bus (1 message) ---
From: codex-main
To: claude-reviewer
Type: review_request
Priority: high
Subject: Review message bus adapter

Request:
Please review the implementation for correctness and lifecycle compatibility.

Acceptance Criteria:
- Verify schema compatibility
- Check no PTY hot path regression
- Confirm CLI JSON output is stable

Artifacts:
- path:docs/2026-07-08-agent-message-bus-design.md
- path:src/main/runtime/orchestration/agent-message-adapter.ts

Risks:
- Might confuse handoff with dispatch

Reply:
orca bus reply --id msg_xxx --summary "..." --request "..." --json

Ack:
orca bus ack --id msg_xxx --note "Accepted" --json

Reject:
orca bus reject --id msg_xxx --note "Reason" --json
---
```

### 13.2 第一阶段不自动注入

第一阶段：

```text
bus message 不自动注入 terminal。
agent 主动运行 orca bus inbox / show。
```

原因：自动注入涉及：

```text
TUI readiness
agent idle detection
terminal send timing
duplicate delivery
read vs delivered semantics
```

这些应放在后续阶段。

### 13.3 后续可选 `--inject`

第二阶段可以支持：

```bash
orca bus send --to <handle> --inject ...
```

实现时应复用 Orca existing delivered_at 和 terminal send 机制。

---

## 14. ArtifactRefs 设计

### 14.1 引用格式

支持：

```text
path:<repo-relative-path>
abs:<absolute-path>
diff:<diff-id>
terminal:<handle>
task:<task-id>
dispatch:<dispatch-id>
pr:<provider>/<owner>/<repo>/<number>
url:<safe-url>
```

第一阶段强制支持：

```text
path:
terminal:
task:
```

其他类型可以保留但不强验证。

### 14.2 Validation

发送时做 best-effort validation：

```text
path exists? warning only
terminal exists? warning only
task exists? warning only
```

不要因为 artifactRef 无法验证就阻止 send，除非用户显式传入 `--strict`。

### 14.3 安全原则

```text
artifactRefs 只是引用，不自动执行。
不要自动打开 URL。
不要自动运行 path 指向的脚本。
不要把 artifact ack 当成 merge/push 授权。
```

---

## 15. 安全设计

### 15.1 Payload 不可信

所有 `payload` 读取必须：

```text
try/catch JSON.parse
schema validation
size limit
unknown fields ignored
invalid envelope treated as legacy message
```

### 15.2 大小限制

建议：

```text
summary <= 500 chars
requestText <= 64 KB
acceptanceCriteria <= 16 KB
risks <= 50 items
artifactRefs <= 100 items
payload JSON <= 256 KB
```

### 15.3 CLI file input

`--request-file` 读取前检查：

```text
file exists
regular file
max size
encoding utf-8
```

### 15.4 不自动执行

Message bus 不应自动执行 message body、artifact path、URL 或 shell command。

### 15.5 不把 ack 当授权

`ack` 只表示 receiver 接受或确认 message，不代表：

```text
允许自动 merge
允许自动 push
允许自动 delete
允许自动 approve PR
允许自动修改文件
```

---

## 16. Migration 策略

### 16.1 Phase 1：无 schema migration

使用现有 `messages.payload`。

优点：

```text
改动小
upstream merge 风险低
方便快速验证协议
```

### 16.2 Phase 2：增加 message_acks

当 ack/reject 需要高效查询或 group ack 时，新增 `message_acks` 表。

### 16.3 Phase 3：可选扩展 messages.type

如果后续希望 DB 层直接支持 `review_request`、`review_response` 等类型，可以扩展 `messages.type` CHECK constraint。

SQLite 修改 CHECK 通常需要 rebuild table。Orca 当前 migration 中已经存在 rebuild messages table 的模式，但仍应谨慎处理，避免 upstream merge 冲突。

---

## 17. 测试计划

### 17.1 Shared unit tests

新增：

```text
src/shared/agent-message-bus/envelope.test.ts
src/shared/agent-message-bus/hash.test.ts
src/shared/agent-message-bus/validation.test.ts
```

覆盖：

```text
valid envelope
invalid envelope
unknown kind
unsupported protocolVersion
hash stable
hash excludes volatile fields
canonical JSON stable key order
payload size limit
```

### 17.2 Adapter tests

新增：

```text
src/main/runtime/orchestration/agent-message-adapter.test.ts
```

覆盖：

```text
envelope -> message insert input
message row -> projection
legacy payload fallback
invalid JSON fallback
review_request coarse type mapping
handoff coarse type mapping
threadId preserved
artifactRefs preserved
```

### 17.3 Service tests

新增：

```text
src/main/runtime/orchestration/agent-message-service.test.ts
```

覆盖：

```text
send structured message
inbox structured message
show message
reply keeps thread
reply reverses from/to
ack updates payload
reject updates payload
ack does not mark read unless explicitly requested
read does not ack
compatibility success
compatibility receiver missing
```

### 17.4 CLI tests

新增：

```text
src/cli/handlers/bus.test.ts
src/cli/specs/bus.test.ts
```

覆盖：

```text
bus send --json
bus inbox --json
bus show --json
bus reply --json
bus ack --json
bus reject --json
bus check-compat --json
request-file
multiple --risk
multiple --artifact
invalid message type
missing receiver
```

### 17.5 Integration tests

后续增加：

```text
Agent A sends review_request to Agent B
Agent B lists inbox
Agent B replies review_response
Agent A sees reply in same thread
Agent A ack
```

---

## 18. 实施计划

### Phase 0 — Baseline review

目标：

```text
确认 fork 与 upstream main 的差异。
确认 orchestration 文件路径。
确认 CLI build/test 命令可运行。
```

输出：

```text
docs/agent-message-bus-baseline.md
```

### Phase 1 — Shared types and adapter

新增：

```text
src/shared/agent-message-bus/types.ts
src/shared/agent-message-bus/envelope.ts
src/shared/agent-message-bus/hash.ts
src/shared/agent-message-bus/validation.ts
src/main/runtime/orchestration/agent-message-adapter.ts
```

完成标准：

```text
Envelope 可创建、校验、hash。
Envelope 可映射到 Orca message insert input。
Orca MessageRow 可投影为 AgentMessageProjection。
Unit tests 通过。
```

### Phase 2 — Service layer

新增：

```text
src/main/runtime/orchestration/agent-message-service.ts
src/main/runtime/orchestration/agent-message-compatibility.ts
```

完成标准：

```text
send/inbox/show/reply/ack/reject/checkCompatibility 可用。
不修改 existing orchestration behavior。
legacy messages 不受影响。
```

### Phase 3 — CLI bus

新增：

```text
src/cli/specs/bus.ts
src/cli/handlers/bus.ts
```

更新 CLI router/help。

完成标准：

```text
orca bus send --json 可发送 structured message。
orca bus inbox --json 可查询。
orca bus reply/ack/reject 可用。
CLI tests 通过。
```

### Phase 4 — Formatter

新增：

```text
src/main/runtime/orchestration/agent-message-formatter.ts
```

完成标准：

```text
orca bus show 默认输出 human-readable format。
orca bus inbox 可显示 concise list。
JSON output 保持稳定。
```

### Phase 5 — ACK table

新增 migration：

```text
message_acks
```

完成标准：

```text
ack/reject 不再只存在 payload。
支持 per-recipient ack。
兼容 Phase 1 payload ack。
```

### Phase 6 — UI Inbox

新增轻量 UI：

```text
Message Inbox panel
Message Detail drawer
Ack/Reject buttons
Artifact refs display
```

完成标准：

```text
用户能在 UI 查看 bus message。
能 ack/reject。
能打开 path artifact。
```

### Phase 7 — Optional injection

增加：

```text
orca bus send --inject
idle-aware delivery
mark delivered once injected
```

完成标准：

```text
不会重复注入。
不会打断 busy TUI。
read/delivered/ack 语义保持分离。
```

---

## 19. 验收标准

第一版完成后应满足：

```text
1. Orca fork 中存在 agent-message-bus shared types。
2. Orca orchestration messages.payload 可保存 structured envelope。
3. CLI 支持 orca bus send/inbox/show/reply/ack/reject/check-compat。
4. Message 可携带 acceptanceCriteria、risks、artifactRefs。
5. Reply 保持 threadId。
6. Ack/reject 不等同于 read/delivered。
7. Handoff 不自动创建 task/dispatch。
8. Review_request 不自动等价于 supervised dispatch。
9. 所有新增逻辑有 unit tests。
10. 原有 orca orchestration tests 不应破坏。
```

---

## 20. 主要风险与缓解

### 20.1 语义混淆

风险：

```text
handoff vs dispatch
read vs ack
review_request vs worker_done
message bus vs orchestration
```

缓解：

```text
CLI 独立命名为 orca bus。
文档明确区分。
不要复用 worker_done 表达普通 review_response。
```

### 20.2 Upstream 同步冲突

风险：Orca 更新频繁，直接大改 orchestration DB 容易冲突。

缓解：

```text
Phase 1 不改 schema。
新文件为主。
少改 existing files。
只在 CLI router/help 做小入口。
```

### 20.3 Agent 误用

风险：agent 把 handoff 当 supervised dispatch。

缓解：

```text
bus formatter 明确提示。
orchestration skill 和 bus skill 分开。
handoff message 不生成 taskId/dispatchId。
```

### 20.4 Payload 膨胀

风险：structured envelope 太大。

缓解：

```text
限制 payload size。
大型内容用 artifactRefs，不直接塞 body。
```

### 20.5 Terminal identity 不稳定

风险：PTY id 可能重启/reattach/SSH relay 后变化。

缓解：

```text
使用 terminal handle 作为 bus session id。
不要用 ptyId 作为长期 message address。
```

---

## 21. 推荐命名

### 21.1 用户可见命名

推荐：

```text
Orca Bus
Agent Message Bus
Structured Messages
```

不推荐：

```text
Forge Bus
Forge Message Bus
```

原因：该 fork 未来应保持 Orca 产品语义。Forge 是设计来源，不应成为用户可见模块名。

### 21.2 内部命名

推荐：

```text
agent-message-bus
AgentMessageEnvelope
AgentMessageService
AgentMessageProjection
AgentMessageCompatibilityReport
```

---

## 22. 最终形态

最终系统应形成：

```text
Orca
  ├─ terminal / worktree / CLI agent runtime
  ├─ existing orchestration task lifecycle
  ├─ mobile / remote / SSH support
  └─ Agent Message Bus
       ├─ structured envelope
       ├─ artifact-aware messages
       ├─ ack / reject
       ├─ review_request / review_response
       ├─ handoff protocol
       └─ compatibility check
```

该方向可以让 Orca fork 获得比 upstream 更强的 agent 协作协议，同时不破坏 upstream Orca 的 runtime、worktree、terminal、mobile 和 orchestration 优势。

---

## 23. 推荐优先级

立即执行：

```text
Phase 1: Shared types and adapter
Phase 2: Service layer
Phase 3: CLI bus
```

暂缓：

```text
ACK table migration
UI inbox
auto injection
orchestration deep linkage
```

第一版应以 CLI + tests 为目标，先证明 structured message bus 能稳定工作，再进入 UI 和深度集成。
