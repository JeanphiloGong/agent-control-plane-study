# Agent 控制面源码学习

先阅读五个产品的真实源码，再讨论各自适合讲解的架构、核心流程和教程版本。
教程范围以源码调查结果及后续讨论为准。

公共教程规划见 [plan.md](plan.md)。

## V0 教程入口

五条教程线的产品总览和模块 V0 设计已经初始化完成。每个产品下面有四个模块，
每个模块包含设计、领域流、状态/错误、接口/存储和参考架构文档。

| 产品 | 总览 |
| --- | --- |
| Herdr | [tutorials/herdr/V0-overview.md](tutorials/herdr/V0-overview.md) |
| Multica | [tutorials/multica/V0-overview.md](tutorials/multica/V0-overview.md) |
| T3 Code | [tutorials/t3-code/V0-overview.md](tutorials/t3-code/V0-overview.md) |
| Lody | [tutorials/lody/V0-overview.md](tutorials/lody/V0-overview.md) |
| Raft | [tutorials/raft/V0-overview.md](tutorials/raft/V0-overview.md) |

## 源码位置

| 项目 | 本地目录 | 上游仓库 |
| --- | --- | --- |
| T3 Code | [sources/t3code](sources/t3code) | [pingdotgg/t3code](https://github.com/pingdotgg/t3code) |
| Raft | [sources/raft-source](sources/raft-source) | [botiverse/raft-source](https://github.com/botiverse/raft-source) |
| Lody | [sources/lody](sources/lody) | [LodyAI/Lody](https://github.com/LodyAI/Lody) |
| Multica | [sources/multica](sources/multica) | [multica-ai/multica](https://github.com/multica-ai/multica) |
| Herdr | [sources/herdr](sources/herdr) | [herdrdev/herdr](https://github.com/herdrdev/herdr) |

每个 `sources/` 子目录都是独立 Git 仓库，保留各自的上游地址和许可证。
克隆参数为 `--depth=1 --filter=blob:none`：取得当前提交的完整工作树，历史较浅；
Git 服务端支持时，未需要的历史文件内容按需获取。

后续 V1 从一个模块的可运行参考实现开始。建议先选择
`herdr-server-client/V1` 或 `multica-scheduler/V1`，完成后再进入对应产品集成门。
