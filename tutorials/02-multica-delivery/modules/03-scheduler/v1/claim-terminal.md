# Multica Scheduler / Lease v1：领取一次计划并记录终态

[V0 设计](../V0-design.md) · [模块路线图](../plan.md) · [本版计划](plan.md) · [下一版 v2](../v2/heartbeat.md)

两个 Scheduler 实例同时发现 10:00 的 `rollup_task_usage_hourly` 已到期。它们尝试领取的是同一次计划执行，由 job 名称、scope 类型、scope ID 和计划时间共同标识。胜者拿到 execution ID 和 lease token，执行完后用这两个值提交结果。真实 job 由 `server/cmd/server/main.go` 注册，`server/internal/scheduler/jobs_task_usage.go` 的 `TaskUsageHourlyJob` 为它指定全局 scope 和三次尝试上限。

## 1. 创建 Go 参考项目

从学习仓库根目录执行。后续版本在同一个 Go module 中继续，需要 Go 1.22 或更新版本。

```bash
mkdir -p tutorials/02-multica-delivery/modules/03-scheduler/v1/reference/scheduler
cd tutorials/02-multica-delivery/modules/03-scheduler/v1/reference
go mod init example.com/multica-scheduler
```

## 2. 定义计划、执行行和领取结果

创建 `v1/reference/scheduler/model.go`。

```go
package scheduler

import (
	"errors"
	"fmt"
	"sync"
	"time"
)

var ErrLeaseLost = errors.New("scheduler: lease lost")

type Status string

const (
	Running Status = "RUNNING"
	Success Status = "SUCCESS"
	Failed  Status = "FAILED"
)

type Plan struct {
	JobName   string
	ScopeKind string
	ScopeID   string
	PlanTime  time.Time
}

type Job struct {
	Plan        Plan
	MaxAttempts int
}

type Execution struct {
	ID          string
	Plan        Plan
	Status      Status
	Attempt     int
	MaxAttempts int
	RunnerID    string
	LeaseToken  string
	StartedAt   time.Time
	UpdatedAt   time.Time
	Result      string
	Error       string
}

type Claim struct {
	Execution  Execution
	Won        bool
	Conflicted bool
}

type Clock interface {
	Now() time.Time
}

type FixedClock struct {
	Current time.Time
}

func (c *FixedClock) Now() time.Time {
	return c.Current
}

type Store struct {
	mu        sync.Mutex
	clock     Clock
	rows      map[Plan]Execution
	nextID    int
	nextToken int
}

func NewStore(clock Clock) *Store {
	return &Store{clock: clock, rows: make(map[Plan]Execution)}
}

func (s *Store) token() string {
	s.nextToken++
	return fmt.Sprintf("lease-%d", s.nextToken)
}
```

`PlanTime` 是调度器已经算好的时间槽。`StartedAt` 记录实际领取时间。10:00 的计划可以在 10:02 领取，两者都会被记录。`Job` 接收已注册的配置，`MaxAttempts` 至少为 1。V3 会在重新领取时读取这个上限。

`LeaseToken` 标识当前尝试。即使同一个 runner 再次领取，也会得到新的 token。示例在 Store 锁内递增计数生成 token，便于测试。Multica 在 PostgreSQL 中用 `gen_random_uuid()` 生成 token。

## 3. 实现领取和两种终态写入

创建 `v1/reference/scheduler/store.go`。调用方把计划和 runner ID 传给 `TryClaim`。拿到 `Won` 的 runner 保存 `Execution.ID` 与 `LeaseToken`，供执行完毕时使用。

```go
package scheduler

import "fmt"

func (s *Store) TryClaim(job Job, runnerID string) Claim {
	s.mu.Lock()
	defer s.mu.Unlock()

	plan := job.Plan
	plan.PlanTime = plan.PlanTime.UTC()
	if _, exists := s.rows[plan]; exists {
		return Claim{Conflicted: true}
	}
	now := s.clock.Now().UTC()
	s.nextID++
	execution := Execution{
		ID:          fmt.Sprintf("execution-%d", s.nextID),
		Plan:        plan,
		Status:      Running,
		Attempt:     1,
		MaxAttempts: job.MaxAttempts,
		RunnerID:    runnerID,
		LeaseToken:  s.token(),
		StartedAt:   now,
		UpdatedAt:   now,
	}
	s.rows[plan] = execution
	return Claim{Execution: execution, Won: true}
}

func (s *Store) FinishSuccess(id, token, result string) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	for plan, execution := range s.rows {
		if execution.ID == id && execution.LeaseToken == token && execution.Status == Running {
			execution.Status = Success
			execution.Result = result
			execution.UpdatedAt = s.clock.Now().UTC()
			s.rows[plan] = execution
			return nil
		}
	}
	return ErrLeaseLost
}

func (s *Store) FinishFailure(id, token, message string) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	for plan, execution := range s.rows {
		if execution.ID == id && execution.LeaseToken == token && execution.Status == Running {
			execution.Status = Failed
			execution.Error = message
			execution.UpdatedAt = s.clock.Now().UTC()
			s.rows[plan] = execution
			return nil
		}
	}
	return ErrLeaseLost
}

func (s *Store) Get(plan Plan) (Execution, bool) {
	s.mu.Lock()
	defer s.mu.Unlock()

	plan.PlanTime = plan.PlanTime.UTC()
	execution, found := s.rows[plan]
	return execution, found
}
```

`TryClaim` 在持锁期间完成查重和插入。`PlanTime` 先转成 UTC，同一时刻用不同时区表示时也会查到同一行。两个终态方法同时检查 ID、token 和 `RUNNING`，因此终态一经写入，重复提交也会得到 `ErrLeaseLost`。`Get` 读取当前行，`Claim.Execution` 则是领取时的副本。

真实源码中，`server/migrations/113_sys_cron_executions.up.sql` 用 `uq_sys_cron_execution` 建立同样的唯一键。`server/internal/scheduler/db_ops.go` 的 `tryClaim` 首先执行 `INSERT … ON CONFLICT DO NOTHING RETURNING …`。两个终态方法把 ID、token 和状态写进 `UPDATE` 的 `WHERE`，用受影响行数判断提交是否生效。示例用单进程 mutex 演示这组原子状态转换。

## 4. 验收同一计划只能领取一次

创建 `v1/reference/scheduler/store_test.go`。测试通过同一个 Store 发起竞争，再把胜者拿到的 ID 和 token 交给终态方法。

```go
package scheduler

import (
	"errors"
	"testing"
	"time"
)

func scheduledJob() Job {
	return Job{
		Plan: Plan{
			JobName:   "rollup_task_usage_hourly",
			ScopeKind: "global",
			ScopeID:   "global",
			PlanTime:  time.Unix(100, 0).UTC(),
		},
		MaxAttempts: 3,
	}
}

func TestOnlyOneWorkerWinsClaim(t *testing.T) {
	job := scheduledJob()
	store := NewStore(&FixedClock{Current: job.Plan.PlanTime})
	start := make(chan struct{})
	claims := make(chan Claim, 2)
	for _, runner := range []string{"worker-a", "worker-b"} {
		go func(runner string) {
			<-start
			claims <- store.TryClaim(job, runner)
		}(runner)
	}
	close(start)
	winner, loser := <-claims, <-claims
	if loser.Won {
		winner, loser = loser, winner
	}
	if !winner.Won || !loser.Conflicted {
		t.Fatalf("winner=%+v loser=%+v", winner, loser)
	}
	if err := store.FinishSuccess(winner.Execution.ID, winner.Execution.LeaseToken, "ok"); err != nil {
		t.Fatal(err)
	}
	row, found := store.Get(job.Plan)
	if !found || row.Status != Success || row.Result != "ok" || row.Attempt != 1 {
		t.Fatalf("stored execution = %+v", row)
	}
}

func TestFinishRejectsWrongTokenAndTerminalRow(t *testing.T) {
	job := scheduledJob()
	store := NewStore(&FixedClock{Current: job.Plan.PlanTime})
	row := store.TryClaim(job, "worker-a").Execution
	if err := store.FinishSuccess(row.ID, "wrong-token", "bad"); !errors.Is(err, ErrLeaseLost) {
		t.Fatalf("wrong token returned %v", err)
	}
	if err := store.FinishFailure(row.ID, row.LeaseToken, "handler failed"); err != nil {
		t.Fatal(err)
	}
	if err := store.FinishSuccess(row.ID, row.LeaseToken, "late result"); !errors.Is(err, ErrLeaseLost) {
		t.Fatalf("terminal write returned %v", err)
	}
	stored, _ := store.Get(job.Plan)
	if stored.Status != Failed || stored.Error != "handler failed" {
		t.Fatalf("stored execution = %+v", stored)
	}
}

func TestPlanIdentityIncludesScopeAndTime(t *testing.T) {
	job := scheduledJob()
	store := NewStore(&FixedClock{Current: job.Plan.PlanTime})
	first := store.TryClaim(job, "worker-a")
	job.Plan.PlanTime = job.Plan.PlanTime.In(time.FixedZone("UTC+8", 8*60*60))
	if claim := store.TryClaim(job, "worker-b"); !claim.Conflicted {
		t.Fatalf("same instant should conflict: %+v", claim)
	}
	job.Plan.ScopeKind = "workspace"
	job.Plan.ScopeID = "other"
	second := store.TryClaim(job, "worker-b")
	job.Plan.PlanTime = job.Plan.PlanTime.Add(time.Minute)
	third := store.TryClaim(job, "worker-b")
	if !first.Won || !second.Won || !third.Won ||
		first.Execution.ID == second.Execution.ID || second.Execution.ID == third.Execution.ID {
		t.Fatalf("plans should have distinct executions: %+v %+v %+v", first, second, third)
	}
}
```

在 `v1/reference/` 内运行以下命令。

```bash
go test ./...
```

预期三个测试通过。真实源码中可以对照 `server/internal/scheduler/concurrent_claim_test.go` 的 `TestConcurrentClaimsSingleWinner`。下一版会在领取时写入心跳和过期时间。
