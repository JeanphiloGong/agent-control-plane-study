# Lody ACP Agent Runtime V0

## 版本与范围

- **源码**：`apps/cli/src/agent/acp-runner.ts`、`managed-agent-runtime.ts`、`agent-client.ts`
- **目标**：把 Machine 上的 Agent 进程和 ACP 协议适配为 Session 可消费的统一 runtime。
- **排除**：Session store、跨会话 Operation 和 E2EE 同步。

## 真实流程

```text
Machine config → acp-runner → managed-agent-runtime
  → AgentClient / ACP transport
  → capability + session events
  → SessionExecutionService
```

`ManagedRuntime*` 负责安装/启动/诊断管理，`AgentClient` 负责 ACP request、notification、steer 和错误转换。Session 上层不能依赖某个 Provider 的原始进程输出。

## 源码对应

| 路径 | 符号/责任 |
| --- | --- |
| `acp-runner.ts` | ACP runner 启动入口和 Agent process 连接 |
| `managed-agent-runtime.ts` | `ManagedRuntime*` 类型、安装、版本和失败分类 |
| `agent-client.ts` | `AgentClient`、transport retry、ACP error 和 session notification |

## 前置、产出与交接

- **前置**：无。它定义机器到 ACP 进程的最小运行时边界。
- **V1 产出**：一个 fake 或本地 ACP process 的启动、事件归一化和结束状态。
- **下游**：[`02-session`](../02-session/V0-design.md) 使用统一 Agent event 管理 Session 生命周期。
- **冻结条件**：Provider 进程退出会产生明确事件，Session 上层不会把连接断开伪造成任务完成。

## 局部版本线

- V1：fake ACP process 启动一个 Session。
- V2：capability、steer/cancel 和 transport retry。
- V3：机器离线、安装失败和 Provider 变化的统一诊断。

## V1 检查

用 fake ACP server 启动、发送 prompt、接收 event。模拟认证错误、method not found 和断线，断言 Session 得到分类错误而不是 Provider-specific 异常。

契约见 `contracts/`，参考架构见 `reference-architecture.md`。
