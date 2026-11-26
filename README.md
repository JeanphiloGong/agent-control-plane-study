# Agent 控制面源码学习

先阅读六个产品的真实源码，再讨论各自适合讲解的架构、核心流程和教程版本。
教程范围以源码调查结果及后续讨论为准。

公共教程规划见 [plan.md](plan.md)。

平台内核与默认应用方案见 [plans/agent-kernel-platform.md](plans/agent-kernel-platform.md)。

统一教程入口见 [tutorials/README.md](tutorials/README.md)。目录编号表示推荐学习顺序，
产品目录内的模块编号表示该产品内部的学习顺序。`V0`、`V1` 等仍然是模块自己的版本，
不是六个产品共享的全局版本。

## V0 教程入口

五条既有教程线的产品总览和模块 V0 设计已经初始化完成。Orca 已完成产品总览和模块路线图，V0 设计仍待构建。每个已完成产品下面有四个模块，
每个模块包含设计、领域流、状态/错误、接口/存储和参考架构文档。

| 产品 | 总览 |
| --- | --- |
| 01 Herdr | [tutorials/01-herdr-runtime/plan.md](tutorials/01-herdr-runtime/plan.md) |
| 02 Multica | [tutorials/02-multica-delivery/V0-overview.md](tutorials/02-multica-delivery/V0-overview.md) |
| 03 T3 Code | [tutorials/03-t3-orchestration/V0-overview.md](tutorials/03-t3-orchestration/V0-overview.md) |
| 04 Lody | [tutorials/04-lody-shared-session/V0-overview.md](tutorials/04-lody-shared-session/V0-overview.md) |
| 05 Raft | [tutorials/05-raft-collaboration/V0-overview.md](tutorials/05-raft-collaboration/V0-overview.md) |
| 06 Orca | [tutorials/06-orca-agent-orchestration/V0-overview.md](tutorials/06-orca-agent-orchestration/V0-overview.md) |

## 源码位置

| 项目 | 本地目录 | 上游仓库 |
| --- | --- | --- |
| T3 Code | [sources/t3code](sources/t3code) | [pingdotgg/t3code](https://github.com/pingdotgg/t3code) |
| Raft | [sources/raft-source](sources/raft-source) | [botiverse/raft-source](https://github.com/botiverse/raft-source) |
| Lody | [sources/lody](sources/lody) | [LodyAI/Lody](https://github.com/LodyAI/Lody) |
| Multica | [sources/multica](sources/multica) | [multica-ai/multica](https://github.com/multica-ai/multica) |
| Herdr | [sources/herdr](sources/herdr) | [herdrdev/herdr](https://github.com/herdrdev/herdr) |
| Orca | [sources/orca](sources/orca) | [stablyai/orca](https://github.com/stablyai/orca) |

每个 `sources/` 子目录都是独立 Git 仓库，保留各自的上游地址和许可证。
克隆参数为 `--depth=1 --filter=blob:none`：取得当前提交的完整工作树，历史较浅；
Git 服务端支持时，未需要的历史文件内容按需获取。

第一条实现路线从 Herdr 的服务端与客户端开始。
[Herdr Server / Client v1–v4](tutorials/01-herdr-runtime/modules/01-server-client/plan.md)
