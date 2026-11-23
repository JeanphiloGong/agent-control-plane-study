# Lody Session Lifecycle V0

## 版本与范围

- **源码**：`session-manager.ts`、`session-execution-service.ts`、`commands/session.ts`
- **目标**：解释 Session 的创建、执行、steer、cancel、archive 和 resume。
- **排除**：ACP adapter 细节、跨 Session Operation 和同步。

## 真实流程

```text
session command → SessionManager
  → SessionExecutionService
  → ACP turn/input
  → completion/cancel/error
  → session record / archive / resume
```

Session 是长期可查询的会话事实，turn 是其中一次执行。`SessionExecutionService` 还处理 pending input、cancellation、handoff 和 finalization，不能把 process exit 直接当成 session archived。

## 源码对应

| 路径 | 符号/责任 |
| --- | --- |
| `session-manager.ts` | Session 记录、查找和生命周期入口 |
| `session-execution-service.ts` | `SessionExecutionService`、turn dispatch、cancel/finalization |
| `commands/session.ts` | CLI/API session 命令入口 |

## 局部版本线

- V1：create/start/read 一个 Session。
- V2：steer、cancel、archive、resume 和 offline 状态。
- V3：turn handoff、pending input 和恢复诊断。

## V1 检查

创建 session，完成一个 fake turn，再执行 cancel/archive/resume。模拟 machine offline，断言 UI/调用方得到不可执行状态。

契约见 `contracts/`，参考架构见 `reference-architecture.md`。
