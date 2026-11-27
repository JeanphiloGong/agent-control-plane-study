# Orca Review / Remote：状态与错误契约

## 状态

~~~text
worker succeeded
  → review-input-ready
  → review-required
  → approved / changes-requested / unverified
  → merged or preserved
  → cleanup eligible / keep
~~~

## 错误边界

- report head 与当前 HEAD 不一致时拒绝 review input。
- host mismatch 拒绝读取，即使 projectId 和 path 相同。
- remote unreachable 返回 unverified，并保持 cleanup keep。
- clean checkout 但 HEAD 未落在 integration ref 时返回 unlanded。
- review-required 不等于 approved，approved 不等于 merged。
- cleanup action delete 之后仍需要 host-local removal settlement。

## 必测状态

1. diff 与绑定的 base/head 一致时才能进入 review-required。
2. 延迟 commit 改变 head 后，旧 review input 失效。
3. 两个 host 同路径只能选中请求的 host-qualified identity。
4. remote 断线后工作区存在且未被清除。
