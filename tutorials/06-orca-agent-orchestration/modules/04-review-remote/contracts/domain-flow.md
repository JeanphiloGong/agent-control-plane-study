# Orca Review / Remote：领域流契约

~~~text
Dispatch settlement + workspace
  → select host and exact workspace identity
  → read base/head/diff and check evidence
  → create review input
  → reachable review or unverified handoff
  → human/external merge decision
  → cleanup evaluation and removal settlement
~~~

| 对象 | 所有者 | 不拥有 |
| --- | --- | --- |
| Review evidence | execution host read | approval 或 merge |
| Host selection | coordinator placement | remote host 的内部进程 |
| Review input | reviewer boundary | cleanup deletion |
| Cleanup verdict | workspace cleanup | review approval |

远程断线的唯一安全结论是本次读取未知。原有 Dispatch、workspace 和可能未提交工作继续保留。
