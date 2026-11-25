# Herdr Remote Endpoint / Plugin v3：manifest 与 invocation context

[模块路线图](../plan.md) · [上一版 v2](../v2/plan.md) · [本版正文](manifest-context.md) · [下一版 v4](../v4/plan.md)

## 版本定位

远程连接已经有明确的 status，但插件命令不能从一份任意字典直接启动。v3 先把 manifest 规范化为稳定的 `PluginManifest`，校验 plugin/action id、版本、命令、平台、重复 action 和 hook 名称，再生成只读 `InvocationContext`。只有通过这些边界的 action 才能交给 v4 runner。

真实源码对应 `src/app/api/plugins/manifest.rs::load_plugin_manifest` 与 `normalize_manifest_*`，以及 `src/app/api/plugins/context.rs::current_plugin_context`、`plugin_context_for_event`。参考项目用 Python dict 表示已解析的 manifest，省略 TOML 解析库和真实 App state。

## 实现顺序

1. 在 `v1/reference/src/plugins/manifest.py` 定义 manifest、action、event hook 和严格规范化函数。
2. 在 `v1/reference/src/plugins/context.py` 定义只读 invocation context、上下文覆盖和 action context 匹配。
3. 用非法 id/version/command/platform、重复 action、未声明 action 和上下文不匹配测试 runner 前拒绝。

## 验收与交接

累计测试应通过 v1–v3。该检查证明非法声明不会抵达命令执行层，合法 action 得到稳定 context。它不证明真实插件文件、manifest TOML、平台版本比较和 App 事件枚举已经在 Python 项目中实现。v4 将这些已验证对象接到 bounded command runner。
