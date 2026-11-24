# Herdr Pane Detection v2：把 blocker 当作证据

[上一版 v1](../v1/pty-detection.md) · [本版计划](plan.md) · [下一版 v3](../v3/plan.md)

v1 的 Unknown 不能解释来源。现在加入 manifest 规则，让 blocker、working 和 idle 都带着可追踪的证据返回。

## 1. 添加 Manifest 与 Detection

在 `v1/reference/src/detection.rs` 中加入：

```rust
#[derive(Clone, Copy, Debug, Eq, PartialEq)]
pub enum EvidenceSource { Screen, Process }

#[derive(Clone, Debug, Eq, PartialEq)]
pub struct AgentDetection { pub state: AgentState, pub source: EvidenceSource, pub visible_blocker: bool }

#[derive(Clone, Debug, Eq, PartialEq)]
pub struct AgentManifest { pub working_marker: String, pub idle_marker: String, pub blocker_marker: String }

pub fn classify(text: &str, manifest: &AgentManifest, process_exited: bool) -> AgentDetection {
    if process_exited { return AgentDetection { state: AgentState::Idle, source: EvidenceSource::Process, visible_blocker: false }; }
    if text.contains(&manifest.blocker_marker) { return AgentDetection { state: AgentState::Blocked, source: EvidenceSource::Screen, visible_blocker: true }; }
    if text.contains(&manifest.working_marker) { return AgentDetection { state: AgentState::Working, source: EvidenceSource::Screen, visible_blocker: false }; }
    if text.contains(&manifest.idle_marker) { return AgentDetection { state: AgentState::Idle, source: EvidenceSource::Screen, visible_blocker: false }; }
    AgentDetection { state: AgentState::Unknown, source: EvidenceSource::Screen, visible_blocker: false }
}
```

将 `AgentState` 增加 `Blocked`。运行 v1 回归和新增 blocker、working、unknown 测试：

```bash
cargo test --manifest-path tutorials/01-herdr-runtime/modules/02-pane-detection/v1/reference/Cargo.toml
```
