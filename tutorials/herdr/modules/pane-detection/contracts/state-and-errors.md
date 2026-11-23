# PTY / Pane / Agent Detection：状态与错误

## 状态规则

- `Unknown` 表示 plain shell 或暂时没有足够证据，不代表 Agent 已完成。
- `Working → Idle` 且没有 visible idle/blocker、Agent 没变、进程没有退出时，先进入 pending idle。
- pending idle 需要多次确认或达到上限后才发布 idle。
- `process_exited` 是独立事实，不能与屏幕上仍可见的 prompt 混为一谈。
- stable idle 且 detection content sequence 未变化时可以跳过昂贵的屏幕读取；Agent 变更、进程退出或内容变化时必须重新读取。

## 失败边界

- PTY read 失败不等于 Agent blocked，应保留可区分的 runtime/terminal 错误。
- manifest 未识别时进入 `Unknown` 或保留已有状态，不能从任意日志文本推断 blocker。
- 检测器跳过状态更新时，调用方仍需执行生命周期和 Agent discovery bookkeeping。
