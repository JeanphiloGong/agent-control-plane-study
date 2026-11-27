# Orca Worktree / Workspace：领域流契约

## 领域操作

```text
create request
  → select project/repo and execution host
  → resolve parent and base ref
  → create or adopt Git worktree
  → stamp instance/host/provenance metadata
  → publish catalog projection

cleanup scan
  → collect activity, Git and runtime evidence
  → classify reasons and blockers
  → evaluate policy and grace window
  → produce cleanup verdict
  → authorize and settle removal separately
```

## 对象所有权

| 对象 | 拥有的事实 | 不拥有的事实 |
| --- | --- | --- |
| CLI selector layer | flags、selector、父子选择和用户错误 | Git checkout、目录删除和最终 cleanup verdict |
| Runtime / execution host | Git worktree 创建、host-local path、filesystem 和 Git scan | 其他 host 的目录或 renderer 的显示状态 |
| `WorktreeMeta` store | Orca 创建 provenance、instance、base、branch protection 和用户 metadata | 当前 Git HEAD 或实时 terminal 是否仍在运行 |
| Git catalog | path、HEAD、branch、main/linked、lock 和 removal 状态 | Orca 是否把 workspace 标为 pinned 或 active |
| Cleanup evaluator | 可解释的 keep/delete/archive decision | 实际 filesystem 删除和远程 host 操作 |

## 关键交接

1. `worktree.create` 完成后，catalog 和 metadata 必须能通过同一 repo、host 和 instance
   关联起来。
2. `worktree.show` 以 catalog Git 事实为基础，再叠加 metadata 和 lineage projection。
3. cleanup scan 的 candidate 必须带有 freshness 信息，evaluation 不能接收“缺失即 clean”。
4. cleanup verdict 传给 removal flow 时必须保留 candidate identity 和 fingerprint，避免
   使用过期扫描结果删除另一 checkout。
