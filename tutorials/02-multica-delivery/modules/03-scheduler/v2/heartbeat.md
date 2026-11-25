# Multica Scheduler / Lease v2：让运行中的 runner 续期

[上一版 v1](../v1/claim-terminal.md) · [本版计划](plan.md) · [模块路线图](../plan.md) · [下一版 v3](../v3/stale-steal.md)

一个 runner 在 10:00 领到计划，设置两分钟的续期窗口。它在 10:00:30 发出心跳，过期时间应从 10:02 推到 10:02:30。`HeartbeatAt` 记录最后一次续期时间，`StaleAfter` 提供后续判断接管的时间点。

以下改动接在 V1 的 `v1/reference/` 上。示例直接调用存储方法并控制时间，真实 Multica 的定时调用位于 `manager.go`。

## 1. 领取时建立续期窗口

在 `v1/reference/scheduler/model.go` 完整替换 `Job`。

```go
type Job struct {
	Plan         Plan
	MaxAttempts  int
	StaleTimeout time.Duration
}
```

同文件的 `Execution` 在 `LeaseToken` 后新增两个字段。`time` 已在 V1 导入。

```go
HeartbeatAt time.Time
StaleAfter  time.Time
```

在 `v1/reference/scheduler/store.go` 完整替换 `Store.TryClaim`。`leaseDeadline` 由紧接的第 2 步实现，两步一起应用后运行验收。

```go
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
		HeartbeatAt: now,
		StaleAfter:  leaseDeadline(now, job.StaleTimeout),
		StartedAt:   now,
		UpdatedAt:   now,
	}
	s.rows[plan] = execution
	return Claim{Execution: execution, Won: true}
}
```

## 2. 用当前 lease 续期

创建 `v1/reference/scheduler/heartbeat.go`。`Heartbeat` 接收领取时拿到的 ID 和 token，以及注册 job 的 `StaleTimeout`。

```go
package scheduler

import "time"

func leaseDeadline(now time.Time, timeout time.Duration) time.Time {
	seconds := max(1, timeout / time.Second)
	return now.Add(seconds * time.Second)
}

func (s *Store) Heartbeat(id, token string, staleTimeout time.Duration) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	for plan, execution := range s.rows {
		if execution.ID == id && execution.LeaseToken == token && execution.Status == Running {
			now := s.clock.Now().UTC()
			execution.HeartbeatAt = now
			execution.StaleAfter = leaseDeadline(now, staleTimeout)
			execution.UpdatedAt = now
			s.rows[plan] = execution
			return nil
		}
	}
	return ErrLeaseLost
}
```

续期窗口从这次心跳的时间重新计算。`leaseDeadline` 和 Multica 的 SQL 调用一样，按整秒计算窗口，最短为一秒。心跳、领取和终态写入共用 Store 的锁，因此校验与写入处于同一个临界区。

对照 `server/internal/scheduler/db_ops.go`。Multica 的 `dbNow` 查询 PostgreSQL 的 `SELECT now()` 供调度和领取使用。`heartbeat` 在 `UPDATE` 内直接用 `now()` 计算新期限。`server/internal/scheduler/manager.go` 的 `runClaimed` 在 handler 执行期间启动 `runHeartbeats`，handler 返回后停止心跳并等待其退出，再尝试写终态。心跳收到 `ErrLeaseLost` 会退出循环，handler 结束后的终态更新仍会检查 token。

## 3. 验收心跳与终态边界

创建 `v1/reference/scheduler/heartbeat_test.go`。

```go
package scheduler

import (
	"errors"
	"testing"
	"time"
)

func TestHeartbeatRenewsOnlyCurrentRunningLease(t *testing.T) {
	job := scheduledJob()
	job.StaleTimeout = 2 * time.Minute
	clock := &FixedClock{Current: job.Plan.PlanTime}
	store := NewStore(clock)
	owner := store.TryClaim(job, "worker-a").Execution
	if !owner.HeartbeatAt.Equal(clock.Current) ||
		!owner.StaleAfter.Equal(clock.Current.Add(job.StaleTimeout)) {
		t.Fatalf("initial lease = %+v", owner)
	}

	clock.Current = clock.Current.Add(30 * time.Second)
	if err := store.Heartbeat(owner.ID, owner.LeaseToken, job.StaleTimeout); err != nil {
		t.Fatal(err)
	}
	renewed, _ := store.Get(job.Plan)
	if !renewed.HeartbeatAt.Equal(clock.Current) ||
		!renewed.StaleAfter.Equal(clock.Current.Add(job.StaleTimeout)) ||
		!renewed.UpdatedAt.Equal(clock.Current) {
		t.Fatalf("renewed lease = %+v", renewed)
	}

	clock.Current = clock.Current.Add(30 * time.Second)
	if err := store.Heartbeat(owner.ID, "wrong-token", job.StaleTimeout); !errors.Is(err, ErrLeaseLost) {
		t.Fatalf("wrong token returned %v", err)
	}
	if stored, _ := store.Get(job.Plan); stored != renewed {
		t.Fatalf("wrong token changed row: %+v", stored)
	}
	if err := store.FinishSuccess(owner.ID, owner.LeaseToken, "ok"); err != nil {
		t.Fatal(err)
	}
	if err := store.Heartbeat(owner.ID, owner.LeaseToken, job.StaleTimeout); !errors.Is(err, ErrLeaseLost) {
		t.Fatalf("terminal row returned %v", err)
	}
}
```

在 `v1/reference/` 内运行以下命令。

```bash
go test ./...
```

预期 V1 的三个测试与心跳测试都通过。这个版本已经记录可供接管判断的 `StaleAfter`。V3 会在竞争同一计划时读取它，并在符合条件时替换 token。
