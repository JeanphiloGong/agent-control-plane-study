# Herdr Pane Detection v4：把多个 Agent 状态投影给客户端

[上一版 v3](../v3/idle-stabilization.md) · [本版计划](plan.md) · [Herdr 路线图](../../plan.md)

多个 Agent 不能共享一个全局 manifest。v4 为每个名称保存独立规则和状态，只投影真正发生变化的事件。

## 1. 添加 Registry

在 `v1/reference/src/detection.rs` 中加入：

```rust
use std::collections::BTreeMap;

#[derive(Clone, Debug, Eq, PartialEq)]
pub struct StatusEvent { pub agent: String, pub state: AgentState, pub revision: u64 }

#[derive(Default)]
pub struct ManifestRegistry {
    manifests: BTreeMap<String, AgentManifest>,
    states: BTreeMap<String, TerminalState>,
}

impl ManifestRegistry {
    pub fn register(&mut self, name: impl Into<String>, manifest: AgentManifest) { self.manifests.insert(name.into(), manifest); }
    pub fn detect(&mut self, name: &str, text: &str, exited: bool) -> Option<StatusEvent> {
        let manifest = self.manifests.get(name)?;
        let detection = classify(text, manifest, exited);
        let state = self.states.entry(name.to_owned()).or_insert_with(TerminalState::new);
        if state.state == detection.state { return None; }
        state.state = detection.state;
        state.revision += 1;
        Some(StatusEvent { agent: name.to_owned(), state: state.state, revision: state.revision })
    }
    pub fn state(&self, name: &str) -> Option<&TerminalState> { self.states.get(name) }
}
```

注册 Codex 和 Claude 两份 manifest，断言规则和 revision 独立。未知 Agent 返回 None，不发送伪造状态。运行完整检测回归：

```bash
cargo test --manifest-path tutorials/01-herdr-runtime/modules/02-pane-detection/v1/reference/Cargo.toml
```

## 检查点与下一步

v4 完成后，检测核心有自己的规则、稳定化和状态投影边界。它可以把状态事件交给 server/client，也可以由 Session Persistence 保存 terminal identity 和 Agent 元数据。真实 manifest 下载、UI 渲染和任务调度由对应模块负责。
