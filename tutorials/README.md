# Agent Control Plane Tutorials

本目录按推荐学习顺序组织五条产品源码教程线。产品目录的编号表示跨产品的学习顺序，
`modules/` 下的编号表示该产品内部的学习顺序。

## Learning Order

| Order | Product line | Entry | Source |
| --- | --- | --- | --- |
| 01 | Herdr agent runtime | [V0 overview](01-herdr-runtime/V0-overview.md) | `sources/herdr` |
| 02 | Multica agent delivery | [V0 overview](02-multica-delivery/V0-overview.md) | `sources/multica` |
| 03 | T3 Code orchestration | [V0 overview](03-t3-orchestration/V0-overview.md) | `sources/t3code` |
| 04 | Lody shared sessions | [V0 overview](04-lody-shared-session/V0-overview.md) | `sources/lody` |
| 05 | Raft collaboration space | [V0 overview](05-raft-collaboration/V0-overview.md) | `sources/raft-source` |

每条产品线的 `V0-overview.md` 先建立真实源码中的业务场景、模块关系和集成门。模块
目录中的 `V0-design.md` 和 `reference-architecture.md` 再展开源码符号、状态边界和
学习实现。模块的 `V1`、`V2` 等版本只属于该模块，产品之间不共享版本号。

## First Checkpoint

从 `01-herdr-runtime/modules/01-server-client` 开始。它先建立服务端、客户端、Socket
握手、命令和事件的基本控制面边界，再进入 PTY、持久化和远程能力。

源码固定提交、调用链和产品集成门见根目录的 [plan.md](../plan.md)。

模块的前置关系、产出契约和产品集成顺序见
[dependency-map.md](dependency-map.md)。先完成一个模块的 V0 设计和 V1 冻结，
再进入它的下游模块；五条产品线之间不共享实现状态。
