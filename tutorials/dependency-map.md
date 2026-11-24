# Module Dependency Map

这张图定义五条产品教程线的学习和实现交接。箭头表示下游模块依赖上游模块已经
冻结的事实或契约。产品线之间没有共享实现状态，产品编号只提供推荐阅读顺序。

## Herdr

```text
01 Server / Client / Socket
  → 02 PTY / Pane / Agent Detection
  → 03 Session Persistence
  → 04 Remote Endpoint / Plugin
  → H1 product integration
```

| Module | Prerequisites | V1 output | Downstream use |
| --- | --- | --- | --- |
| `01-server-client` | None | Server-owned runtime, handshake, client command boundary, disconnect isolation | Pane runtime and client event projection |
| `02-pane-detection` | `01-server-client` runtime/event ownership | Terminal evidence, `AgentDetection`, stabilized agent state event | Session snapshot and status projection |
| `03-session-persistence` | `01-server-client` runtime ownership and `02-pane-detection` terminal state | Versioned snapshot capture, load, restore and failure boundary | H1 recovery and later remote handoff |
| `04-remote-plugins` | `01-server-client` endpoint boundary and `03-session-persistence` identity rules | Endpoint transport state and constrained plugin invocation | H1 remote/plugin integration |

## Multica

```text
01 Issue / Invocation
  → 02 Runtime Access
  → 03 Scheduler / Lease
  → 04 Failure / Review
  → M1 product integration
```

| Module | Prerequisites | V1 output | Downstream use |
| --- | --- | --- | --- |
| `01-issue-invocation` | None | Authorized `RunRequest` with no-start rejection paths | Runtime lookup and readiness |
| `02-runtime-access` | `01-issue-invocation` authorization result | Runtime ownership/readiness verdict and runner start boundary | Scheduler claim and run execution |
| `03-scheduler` | `01-issue-invocation` run identity and `02-runtime-access` runtime boundary | Claim, lease token, heartbeat and fenced terminal update | Failure classification and review |
| `04-failure-review` | `03-scheduler` terminal result and lease outcome | Failure class, retry/review decision and Issue projection | M1 end-to-end delivery |

## T3 Code

```text
01 Command / Orchestrator
  → 02 Event / Projection
  → 03 Provider / Effects
  → 04 Workspace / Remote
  → T1 product integration
```

| Module | Prerequisites | V1 output | Downstream use |
| --- | --- | --- | --- |
| `01-orchestrator` | None | Command receipt and one idempotent thread/run decision | Event and projection writes |
| `02-event-projection` | `01-orchestrator` command decision | Durable event, projection and cursor contract | Provider result ingress and client reads |
| `03-provider-effects` | `01-orchestrator` run identity and `02-event-projection` event contract | Provider adapter, outbox effect and result ingress | Workspace checkpoint and remote stream |
| `04-workspace-remote` | `02-event-projection` read model and `03-provider-effects` effect boundary | Checkpoint/rewind and reconnectable client projection | T1 end-to-end Provider turn |

## Lody

```text
01 ACP Runtime
  → 02 Session
  → 03 Operation / Worktree
  → 04 Sync / E2EE
  → L1 product integration
```

| Module | Prerequisites | V1 output | Downstream use |
| --- | --- | --- | --- |
| `01-acp-runtime` | None | Machine-bound ACP process and normalized agent event | Session lifecycle |
| `02-session` | `01-acp-runtime` event and process boundary | Session create/run/steer/cancel/archive/resume lifecycle | Child operations and worktrees |
| `03-operation-worktree` | `02-session` lifecycle and worktree boundary | Parent/child operation, deadline and isolated directory result | Sync and aggregate state |
| `04-sync-e2ee` | `02-session` document identity and `03-operation-worktree` result identity | Scoped document frame, device boundary and cursor recovery | L1 cross-machine session |

## Raft

```text
01 Identity / Workspace
  → 02 Channel / Task
  → 03 Daemon / Credentials
  → 04 Memory / Evidence / Sync
  → R1 product integration
```

| Module | Prerequisites | V1 output | Downstream use |
| --- | --- | --- | --- |
| `01-identity-workspace` | None | Agent identity, workspace membership and capability scope | Message visibility and task claim |
| `02-channel-task` | `01-identity-workspace` scope and membership | Channel/thread visibility, task claim and handoff reference | Daemon dispatch and evidence ownership |
| `03-daemon-credentials` | `01-identity-workspace` capability scope and `02-channel-task` task identity | Machine process lifecycle and scoped credential request | Evidence/context projection |
| `04-memory-evidence-sync` | `02-channel-task` visibility and `03-daemon-credentials` activity boundary | Evidence receipt, context view and cursor-based sync | R1 collaboration flow |

## Version Rule

每个模块按自己的版本线推进：

```text
V0 design and contracts
  → V1 minimum executable boundary
  → V2 state, persistence or concurrency
  → V3 external effects or collaboration
  → product integration gate
```

当前仓库已经完成全部模块的 V0 设计、契约和参考架构。V1 仍按上表从每条产品线的
第一个模块开始构建。V0 文档中的 V1 检查是实现验收条件，不代表当前已经存在可运行
的参考实现。
