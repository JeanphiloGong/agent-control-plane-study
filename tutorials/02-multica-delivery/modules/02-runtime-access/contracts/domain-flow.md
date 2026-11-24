# Runtime Access：领域流

```text
Agent → RuntimeLookup → ownership/access policy → runtime availability
  → RunnerAdapter.start → Run process
```

Runtime registry 只描述宿主事实，adapter 才产生进程副作用。服务层不能把 registry 记录当成已经启动的 Agent。
