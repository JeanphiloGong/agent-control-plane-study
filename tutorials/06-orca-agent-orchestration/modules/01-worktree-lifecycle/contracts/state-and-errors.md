# Orca Worktree / Workspace：状态与错误契约

## Worktree 身份状态

```text
requested
  → creating
  → registered + metadata persisted
  → observed
  → active / inactive / archived
  → removal requested
  → removing
  → removed or removal-error/leftover
```

`active`、`inactive` 和 `archived` 是投影或 cleanup 输入，不是允许任意删除的权限。一个
workspace 只有在 Git catalog、metadata 和 host-local filesystem 事实一致时，才可以进入
可删除流程。

## Cleanup verdict

| 输入边界 | 结果 |
| --- | --- |
| main worktree 或 folder repo | `keep / not-removable` |
| project cleanup off 或所有规则关闭 | `keep / project-off` 或 `rules-off` |
| pinned、active workspace、live agent、running terminal | `keep`，保留相应 blocker |
| Git 未检查、检查时间为空或 status error | `keep / git-unknown` |
| 规则未到期 | `keep / waiting` 或 `no-rule-matched` |
| 规则命中且最近 activity grace window 未结束 | `keep / recently-active` |
| 规则命中且无保护 | `delete` 或 `archive`，取决于 inactive policy 和 archive capability |

## 错误边界

- default base 查询失败是可诊断错误，只有可用的 persisted base 才能 fallback。
- selector 不明确或不存在时，CLI 不能猜测另一个 worktree。
- 主 worktree、repo root、home root 和危险路径拒绝 removal。
- 未注册 worktree path 拒绝 removal，除非 orphan proof 明确证明它属于该 repo，且 host
  home authority 已解析。
- nested registered worktree 存在时，外层目录拒绝 removal，避免 Git 删除未选中的 checkout。
- 远程或 host 状态未知时，不能把不可读取当成 clean 或 completed。

## 不变量检查

V1 至少要测试以下边界：

1. 同一个 path 的不同 `instanceId` 产生不同 identity key。
2. `resolveWorktreeCreateBase` 在 lookup throw、lookup null、persisted usable 和 explicit
   requested 四种输入下返回不同且可解释的结果。
3. `findRegisteredDeletableWorktree` 拒绝 main、unregistered 和 nested worktree。
4. cleanup candidate 的 `git.clean = null` 进入保护分组，不进入 `nothing-local`。
