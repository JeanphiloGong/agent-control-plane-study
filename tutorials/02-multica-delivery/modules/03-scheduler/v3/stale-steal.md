# Multica Scheduler / Lease v3：接管旧尝试并保护新结果

[上一版 v2](../v2/heartbeat.md) · [本版计划](plan.md) · [模块路线图](../plan.md)

A 领取了 10:00 的计划并停止发送心跳。它的 `StaleAfter` 是 10:02。时钟到了 10:02:01，B 和 C 都来竞争，只有其中一个能在同一行上开启第 2 次尝试。A 可能还在跑，直到它提交结果时，Store 才用新 token 判断这次写入是否属于当前尝试。

以下改动接在 V2 的 `v1/reference/` 上。`TryClaim` 处理已经算好的计划时间。真实 Multica 还会在每个 tick 中规划到期时间槽并清扫遗留执行行。

## 1. 记录接管策略与重试时间

在 `v1/reference/scheduler/model.go` 完整替换 `Job` 和 `Claim`。

```go
type Job struct {
	Plan              Plan
	MaxAttempts       int
	StaleTimeout      time.Duration
	AllowStaleReentry bool
}

type Claim struct {
	Execution  Execution
	Won        bool
	Stole      bool
	Conflicted bool
}
```

同文件的 `Execution` 在 `StaleAfter` 后新增字段。

```go
NextRetryAt time.Time
```

`AllowStaleReentry` 控制对过期 `RUNNING` 行的直接接管。`NextRetryAt` 记录 `FAILED` 行的最早重试时间。零值对应 SQL 的 `NULL`，表示这条记录没有时间上的等待要求，尝试次数上限仍然生效。和真实 `claim` 一样，`Stole` 表示复用行成功，覆盖过期接管和失败重试两种情况。

## 2. 在同一个临界区内判断并重新领取

在 `v1/reference/scheduler/store.go`，把 `import "fmt"` 替换为以下导入，再完整替换 `Store.TryClaim`。

```go
import (
	"fmt"
	"time"
)
```

```go
func (s *Store) TryClaim(job Job, runnerID string) Claim {
	s.mu.Lock()
	defer s.mu.Unlock()

	plan := job.Plan
	plan.PlanTime = plan.PlanTime.UTC()
	now := s.clock.Now().UTC()
	previous, exists := s.rows[plan]
	if exists {
		canSteal := previous.Status == Running && job.AllowStaleReentry &&
			now.After(previous.StaleAfter)
		canRetry := previous.Status == Failed &&
			(previous.NextRetryAt.IsZero() || !now.Before(previous.NextRetryAt))
		if previous.Attempt >= previous.MaxAttempts || (!canSteal && !canRetry) {
			return Claim{Conflicted: true}
		}
	} else {
		s.nextID++
		previous.ID = fmt.Sprintf("execution-%d", s.nextID)
		previous.MaxAttempts = job.MaxAttempts
	}

	execution := Execution{
		ID:          previous.ID,
		Plan:        plan,
		Status:      Running,
		Attempt:     previous.Attempt + 1,
		MaxAttempts: previous.MaxAttempts,
		RunnerID:    runnerID,
		LeaseToken:  s.token(),
		HeartbeatAt: now,
		StaleAfter:  leaseDeadline(now, job.StaleTimeout),
		StartedAt:   now,
		UpdatedAt:   now,
	}
	s.rows[plan] = execution
	return Claim{Execution: execution, Won: !exists, Stole: exists}
}
```

新行从 `Attempt = 1` 开始。复用行沿用 ID 和先前保存的 `MaxAttempts`，用新 token、新 runner 和新的续期窗口启动下一次尝试。示例构造的新值把旧尝试的 `Result`、`Error` 和 `NextRetryAt` 恢复为零值。并发竞争者在同一把锁下读取这次更新，看到的是已经续期的新 lease。

| 当前行 | 能否领取 |
| --- | --- |
| 无记录 | 首次领取，返回 `Won` |
| `RUNNING` | 允许接管、`now > stale_after` 且还有剩余尝试，返回 `Stole` |
| `FAILED` | `now >= next_retry_at` 或无等待时间，且还有剩余尝试，返回 `Stole` |
| `SUCCESS` 或其余情况 | 返回 `Conflicted` |

恰好等于 `StaleAfter` 时还不能接管。到期条件决定何时可以替换 token，旧 runner 的写入仍由当前行的 ID、token 和状态决定。这个检查保护执行记录。handler 的外部副作用需要由具体 job 保证可以安全重入。

源码在 `server/internal/scheduler/db_ops.go` 的 `tryClaim` 中先尝试插入，冲突后用带条件的单条 `UPDATE … RETURNING` 实现重新领取。该 SQL 清除旧错误、完成时间和重试时间。`server/internal/scheduler/manager.go` 的 `processPlan` 只在 `Won` 或 `Stole` 后调用 `runClaimed`。`runJob` 还会用 `markStaleAsFailed` 把遗留的过期行标成 `FAILED`，是否重试该时间槽由后续计划计算和失败领取条件决定。

## 3. 在失败终态中保存调用方给出的重试时间

在 `v1/reference/scheduler/store.go` 完整替换 `Store.FinishFailure`。

```go
func (s *Store) FinishFailure(id, token, message string, nextRetryAt time.Time) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	for plan, execution := range s.rows {
		if execution.ID == id && execution.LeaseToken == token && execution.Status == Running {
			execution.Status = Failed
			execution.Error = message
			execution.NextRetryAt = nextRetryAt.UTC()
			execution.UpdatedAt = s.clock.Now().UTC()
			s.rows[plan] = execution
			return nil
		}
	}
	return ErrLeaseLost
}
```

在 `v1/reference/scheduler/store_test.go` 的 `TestFinishRejectsWrongTokenAndTerminalRow` 中，将 `FinishFailure` 调用所在的 `if` 块替换为以下代码。该文件已导入 `time`。

```go
if err := store.FinishFailure(row.ID, row.LeaseToken, "handler failed", time.Time{}); err != nil {
	t.Fatal(err)
}
```

在真实入口 `runClaimed` 中，handler 失败后由 manager 根据 attempt 选择 backoff，再把算出的绝对时间传给 `finishFailure`。示例测试直接传绝对时间，可以精确检查边界。

## 4. 验收重新领取和旧 token 围栏

创建 `v1/reference/scheduler/stale_steal_test.go`。

```go
package scheduler

import (
	"errors"
	"testing"
	"time"
)

func TestStaleStealRejectsOldLease(t *testing.T) {
	job := scheduledJob()
	job.StaleTimeout = time.Minute
	job.AllowStaleReentry = true
	job.MaxAttempts = 2
	clock := &FixedClock{Current: job.Plan.PlanTime}
	store := NewStore(clock)
	old := store.TryClaim(job, "worker-a").Execution

	clock.Current = old.StaleAfter
	if claim := store.TryClaim(job, "worker-b"); !claim.Conflicted {
		t.Fatalf("lease is still current at equality: %+v", claim)
	}
	clock.Current = old.StaleAfter.Add(time.Second)
	start := make(chan struct{})
	claims := make(chan Claim, 2)
	for _, runner := range []string{"worker-b", "worker-c"} {
		go func(runner string) {
			<-start
			claims <- store.TryClaim(job, runner)
		}(runner)
	}
	close(start)
	winner, loser := <-claims, <-claims
	if loser.Stole {
		winner, loser = loser, winner
	}
	current := winner.Execution
	if !winner.Stole || !loser.Conflicted || current.ID != old.ID ||
		current.RunnerID == old.RunnerID || current.Attempt != 2 ||
		current.LeaseToken == old.LeaseToken ||
		!current.HeartbeatAt.Equal(clock.Current) ||
		!current.StaleAfter.Equal(clock.Current.Add(job.StaleTimeout)) {
		t.Fatalf("old=%+v winner=%+v loser=%+v", old, winner, loser)
	}
	if err := store.Heartbeat(old.ID, old.LeaseToken, job.StaleTimeout); !errors.Is(err, ErrLeaseLost) {
		t.Fatalf("old heartbeat: %v", err)
	}
	if err := store.FinishSuccess(old.ID, old.LeaseToken, "old"); !errors.Is(err, ErrLeaseLost) {
		t.Fatalf("old success: %v", err)
	}
	if err := store.FinishFailure(old.ID, old.LeaseToken, "old", time.Time{}); !errors.Is(err, ErrLeaseLost) {
		t.Fatalf("old failure: %v", err)
	}
	if row, _ := store.Get(job.Plan); row != current {
		t.Fatalf("old token updated the row: %+v", row)
	}

	clock.Current = current.StaleAfter.Add(time.Second)
	if claim := store.TryClaim(job, "worker-d"); !claim.Conflicted {
		t.Fatalf("attempt limit should block another steal: %+v", claim)
	}
	if err := store.FinishSuccess(current.ID, current.LeaseToken, "current"); err != nil {
		t.Fatal(err)
	}
	if row, _ := store.Get(job.Plan); row.Status != Success || row.Result != "current" {
		t.Fatalf("stored execution = %+v", row)
	}
}

func TestStaleStealRequiresReentryPolicy(t *testing.T) {
	job := scheduledJob()
	job.StaleTimeout = time.Minute
	clock := &FixedClock{Current: job.Plan.PlanTime}
	store := NewStore(clock)
	first := store.TryClaim(job, "worker-a").Execution
	clock.Current = first.StaleAfter.Add(time.Second)
	if claim := store.TryClaim(job, "worker-b"); !claim.Conflicted {
		t.Fatalf("stale reentry disabled: %+v", claim)
	}
}

func TestFailedRetryWaitsForDueTimeAndRespectsLimit(t *testing.T) {
	job := scheduledJob()
	job.StaleTimeout = time.Minute
	job.MaxAttempts = 2
	clock := &FixedClock{Current: job.Plan.PlanTime}
	store := NewStore(clock)
	first := store.TryClaim(job, "worker-a").Execution
	due := clock.Current.Add(time.Minute)
	if err := store.FinishFailure(first.ID, first.LeaseToken, "temporary failure", due); err != nil {
		t.Fatal(err)
	}
	if claim := store.TryClaim(job, "worker-b"); !claim.Conflicted {
		t.Fatalf("retry before due time: %+v", claim)
	}

	clock.Current = due
	retry := store.TryClaim(job, "worker-b")
	row := retry.Execution
	if !retry.Stole || row.ID != first.ID || row.Attempt != 2 || row.Status != Running ||
		row.LeaseToken == first.LeaseToken || row.Error != "" || !row.NextRetryAt.IsZero() {
		t.Fatalf("retry = %+v", retry)
	}
	if err := store.FinishFailure(row.ID, row.LeaseToken, "attempts exhausted", time.Time{}); err != nil {
		t.Fatal(err)
	}
	if claim := store.TryClaim(job, "worker-c"); !claim.Conflicted {
		t.Fatalf("attempt limit should block another retry: %+v", claim)
	}
}
```

在 `v1/reference/` 内运行以下命令。

```bash
go test ./...
```

预期 V1 到 V3 的七个测试全部通过。接管测试固定时钟到两个边界点，验证续期、一次接管和旧写入被拒绝。重试测试把时钟推进到刚好到期，验证同一 execution 进入第 2 次尝试。

## 5. 对照真实 Scheduler 的 PostgreSQL 验收

在 `sources/multica/server/` 内，将 `DATABASE_URL` 设为已创建的一次性 PostgreSQL 测试库连接参数。下面的命令在空库上安装执行表及后续索引变更，再运行源码测试。测试会创建并清理自己的执行行。

```bash
: "${DATABASE_URL:?请设置一次性 PostgreSQL 测试库的连接参数}"
export DATABASE_URL
psql "$DATABASE_URL" -v ON_ERROR_STOP=1 \
  -f migrations/113_sys_cron_executions.up.sql \
  -f migrations/301_drop_redundant_sys_cron_job_plan_index.up.sql &&
go test ./internal/scheduler \
  -run '^(TestConcurrentClaimsSingleWinner|TestStaleStealTerminalUpdateIgnored|TestManagerEveryPlanRetriesFailedSamePlanTime)$' \
  -count=1 -v
```

预期三个测试显示 `PASS`。数据库不可达时，这些测试会输出 `SKIP`。重点对照真实测试如何用一条 PostgreSQL 更新模拟过期窗口，再检查旧 token 影响零行。

下一模块基于持久化的运行结果讲失败分类、唤醒与交付审核。 [Failure / Review 的 V0 设计](../../04-failure-review/V0-design.md)
