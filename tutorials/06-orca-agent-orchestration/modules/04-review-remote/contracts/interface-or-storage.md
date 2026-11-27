# Orca Review / Remote：接口与存储契约

## 输入

- Dispatch、Task、workspace identity、hostId 和 worker report
- base commit、head commit、Git diff 和检查结果
- projectId、repo selector、host selection 和远程 RPC receipt
- cleanup candidate 与 merge evidence

## 输出

- Review input，包括 dispatchId、taskId、workspaceKey、base、head、diff 和 checks
- remote reachable、unverified 或 host mismatch
- review/merge 需要的外部链接和 provider metadata
- cleanup 的 merged、unlanded、host-unreachable、dirty 或 keep verdict

## 边界

- Git diff 读取必须指定 worktree、headOid、mergeBase 和 file path。
- provider link 只提供 Review 入口，不能改写 worker outcome。
- host selection 使用 host-qualified identity，不用裸 path 或 display name。
- cleanup removal 在执行时重新加载 candidate，过期 review 输入不能授权删除。
