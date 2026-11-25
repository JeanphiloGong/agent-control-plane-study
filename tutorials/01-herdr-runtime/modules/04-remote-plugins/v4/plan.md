# Herdr Remote Endpoint / Plugin v4：bounded command runner 与状态投影

[模块路线图](../plan.md) · [上一版 v3](../v3/plan.md) · [本版正文](command-runner.md)

## 版本定位

v3 已经保证只有合法 manifest 和匹配 context 才能得到 action。v4 把这个 action 接到命令执行边界：限制并发数和 stdout/stderr 大小，生成受控环境变量，记录 Running/Succeeded/Failed 日志，并把 endpoint/plugin 结果转换成客户端可读的 projection。remote plugin 仍由 manifest 的 source 和 plugin root 标识，不能直接拥有核心 workspace 状态。

真实源码对应 `src/app/api/plugins/runtime.rs::App::start_plugin_command`、`run_plugin_startup_hooks`、`run_plugin_event_hooks`、`read_capped_plugin_output`，以及 `src/client/endpoint/registry.rs` 的 endpoint 状态投影路径。

## 实现顺序

1. 在 `v1/reference/src/plugins/runner.py` 增加可注入 executor、command log、环境边界、输出截断和并发上限。
2. 在 `v1/reference/src/endpoints/projection.py` 增加 endpoint 与 plugin log 的只读客户端 projection。
3. 用嵌套 fake executor 触发并发上限，用失败 executor 检查错误日志和输出上限，用 event hook 检查未知事件不会启动命令。

## 验收与边界

从 v1/reference 执行累计测试，预期所有版本测试通过。检查证明 runner 的副作用边界和客户端可读结果，不证明真实子进程线程、SSH bridge、平台 env 注入、AppEvent 通道和 Unix socket 已经在 Python 项目中运行。Herdr 真实运行时还会回收 in-flight 计数、保留有限日志并读取两个输出 pipe。
