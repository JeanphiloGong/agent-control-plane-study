# Orca Session / Recovery：领域流契约

~~~text
Dispatch + session identity
  → observe host liveness
  → live / exited / unknown
  → inspect exact Dispatch
  → confirmed stop or preserve
  → same-Task retry when cause changed
  → cleanup candidate receives worker/terminal evidence
~~~

| 对象 | 所有者 | 不拥有 |
| --- | --- | --- |
| Session liveness | execution host observation | Task success |
| Stop receipt | stop mutation boundary | Physical exit proof when unknown |
| Recovery decision | owning Run/coordinator | Filesystem deletion |
| Cleanup blocker | Worktree cleanup evaluator | Retry scheduling |

silence、timeout 和 transport disconnect 只产生 unknown。只有 host 对 exact identity 的
明确 exited 观测才产生 exited。
