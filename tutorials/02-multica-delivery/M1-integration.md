# Multica M1：把 Issue、Runtime、Scheduler 和 Review 串成一次 Run

[返回 M1 计划](M1-plan.md) · [产品 V0 总览](V0-overview.md) · [Issue / Invocation](modules/01-issue-invocation/V0-design.md) · [Runtime Access](modules/02-runtime-access/V0-design.md) · [Scheduler / Lease](modules/03-scheduler/V0-design.md) · [Failure / Review](modules/04-failure-review/V0-design.md)

## 版本定位

前四个模块已经分别冻结了权限、Runtime readiness、lease 和失败决策。现在需要回答一个产品问题：

> 一个用户创建的 Issue，经过这些边界后，能否完成一次带 lease 的 Run，并把结果准确投影回 Issue？

本版建立一个小型 Go reference，把四个模块的交接放在同一个 Coordinator 中。读者会看到授权检查在 claim 前结束，Runtime offline 不会产生执行行，成功 Run 进入 in_review，失败 Run 产生 retry 或 Review 决策，旧 lease 无法覆盖新 owner。

## 前置条件

- 已检出 `sources/multica` @ `4ceffd536603c67620c7460f6c412dcbf092299e`，并从学习仓库根目录开始操作。
- 本地安装 Go。M1 reference 不依赖 Multica 的构建或 Provider。
- 只有运行真实 Service 和 Scheduler 对照测试时，才需要一次性的 PostgreSQL 测试数据库，并通过 `DATABASE_URL` 提供连接。

真实 Multica 对照路径如下：

| 集成责任 | 真实入口 |
| --- | --- |
| Issue 写入 | server/internal/service/issue.go:IssueService.Create |
| Invocation policy | server/internal/service/agent_invocation.go:CanMemberInvokeAgent |
| Runtime readiness | server/internal/service/agent_ready.go:AgentReadiness |
| Runtime lookup | server/internal/service/agent_runtime_lookup.go:RuntimeLookup.Get |
| Claim / heartbeat | server/internal/scheduler/manager.go:processPlan、runHeartbeats |
| Lease fencing | server/internal/scheduler/db_ops.go:tryClaim、heartbeat、finishSuccess、finishFailure |
| Run terminal write | server/internal/service/task.go:CompleteTaskWithTransition、FailTaskWithTransition |
| Retry / Review decision | server/internal/service/task.go:retryableReasons、retryAttemptCeiling 和 04-failure-review |

参考项目使用内存 Store 和可控时钟，真实 PostgreSQL 事务仍由生产源码拥有。`Start` 代表 claim 时刻的集成门，真实服务仍可能在 Runtime offline 时保留一个等待中的 queued task。

## 1. 创建集成 reference

在学习仓库根目录执行：

~~~bash
mkdir -p tutorials/02-multica-delivery/M1/reference/delivery
cd tutorials/02-multica-delivery/M1/reference
go mod init example.com/multica-m1
~~~

创建 M1/reference/delivery/model.go：

~~~go
package delivery

import (
	"errors"
	"sync"
	"time"
)

type IssueStatus string

const (
	IssueTodo       IssueStatus = "todo"
	IssueInProgress IssueStatus = "in_progress"
	IssueInReview   IssueStatus = "in_review"
	IssueBlocked    IssueStatus = "blocked"
)

type RuntimeStatus string

const (
	RuntimeOnline  RuntimeStatus = "online"
	RuntimeOffline RuntimeStatus = "offline"
)

type RunStatus string

const (
	RunRunning   RunStatus = "running"
	RunSucceeded RunStatus = "succeeded"
	RunFailed    RunStatus = "failed"
)

type LeaseStatus string

const (
	LeaseRunning LeaseStatus = "RUNNING"
	LeaseDone    LeaseStatus = "DONE"
	LeaseFailed  LeaseStatus = "FAILED"
)

type Issue struct {
	ID       string
	OwnerID  string
	AgentID  string
	Status   IssueStatus
	LastRun  string
}

type Agent struct {
	ID        string
	OwnerID   string
	RuntimeID string
}

type Runtime struct {
	ID     string
	Status RuntimeStatus
}

type Run struct {
	ID          string
	IssueID     string
	AgentID     string
	Status      RunStatus
	Attempt     int
	MaxAttempts int
	Error       string
	Output      string
}

type Execution struct {
	ID         string
	RunID      string
	RunnerID   string
	LeaseToken string
	Status     LeaseStatus
	Heartbeat  time.Time
	StaleAfter time.Time
}

type Outcome struct {
	Issue          Issue
	Run            Run
	Retryable      bool
	ReviewRequired bool
}

var (
	ErrInvocationDenied = errors.New("invocation denied")
	ErrRuntimeWaitable  = errors.New("runtime is offline")
	ErrLeaseLost        = errors.New("lease lost")
	ErrNotFound         = errors.New("object not found")
	ErrLeaseCurrent     = errors.New("lease is still current")
)

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
	mu         sync.Mutex
	clock      Clock
	issues     map[string]Issue
	agents     map[string]Agent
	runtimes   map[string]Runtime
	runs       map[string]Run
	executions map[string]Execution
	nextID     int
	nextToken  int
}

func NewStore(clock Clock) *Store {
	return &Store{
		clock:      clock,
		issues:     map[string]Issue{},
		agents:     map[string]Agent{},
		runtimes:   map[string]Runtime{},
		runs:       map[string]Run{},
		executions: map[string]Execution{},
	}
}

func (s *Store) AddIssue(issue Issue) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.issues[issue.ID] = issue
}

func (s *Store) AddAgent(agent Agent) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.agents[agent.ID] = agent
}

func (s *Store) AddRuntime(runtime Runtime) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.runtimes[runtime.ID] = runtime
}

func (s *Store) Issue(id string) (Issue, bool) {
	s.mu.Lock()
	defer s.mu.Unlock()
	issue, ok := s.issues[id]
	return issue, ok
}

func (s *Store) Run(id string) (Run, bool) {
	s.mu.Lock()
	defer s.mu.Unlock()
	run, ok := s.runs[id]
	return run, ok
}

func (s *Store) Execution(runID string) (Execution, bool) {
	s.mu.Lock()
	defer s.mu.Unlock()
	execution, ok := s.executions[runID]
	return execution, ok
}

func (s *Store) next(prefix string) string {
	s.nextID++
	return prefix + "-" + string(rune('0'+s.nextID))
}

func (s *Store) token() string {
	s.nextToken++
	return "lease-" + string(rune('0'+s.nextToken))
}
~~~

Issue 保存用户可见状态，Run 保存一次 Agent 执行，Execution 保存 Scheduler 的当前 owner。三者分别拥有自己的状态。Outcome 是 Coordinator 返回给上层的查询结果，它同时携带 Issue、Run 和策略决策。

真实服务中这些事实分散在 Issue、agent_task_queue 和 sys_cron_executions 表中。M1 reference 把它们放到一个 Store 中，方便验证交接顺序。

## 2. 实现一个完整 Coordinator

创建 M1/reference/delivery/coordinator.go：

~~~go
package delivery

import (
	"strings"
	"time"
)

type Coordinator struct {
	store       *Store
	staleWindow time.Duration
}

func NewCoordinator(store *Store, staleWindow time.Duration) *Coordinator {
	return &Coordinator{store: store, staleWindow: staleWindow}
}

func (c *Coordinator) Start(issueID, memberID, runnerID string) (Run, Execution, error) {
	c.store.mu.Lock()
	defer c.store.mu.Unlock()

	issue, ok := c.store.issues[issueID]
	if !ok {
		return Run{}, Execution{}, ErrNotFound
	}
	agent, ok := c.store.agents[issue.AgentID]
	if !ok {
		return Run{}, Execution{}, ErrNotFound
	}
	if issue.OwnerID != memberID || agent.OwnerID != memberID {
		return Run{}, Execution{}, ErrInvocationDenied
	}
	runtime, ok := c.store.runtimes[agent.RuntimeID]
	if !ok {
		return Run{}, Execution{}, ErrNotFound
	}
	if runtime.Status != RuntimeOnline {
		return Run{}, Execution{}, ErrRuntimeWaitable
	}

	run := Run{
		ID:          c.store.next("run"),
		IssueID:     issue.ID,
		AgentID:     agent.ID,
		Status:      RunRunning,
		Attempt:     1,
		MaxAttempts: 2,
	}
	now := c.store.clock.Now().UTC()
	execution := Execution{
		ID:         c.store.next("execution"),
		RunID:      run.ID,
		RunnerID:   runnerID,
		LeaseToken: c.store.token(),
		Status:     LeaseRunning,
		Heartbeat:  now,
		StaleAfter: now.Add(c.staleWindow),
	}
	issue.Status = IssueInProgress
	issue.LastRun = run.ID
	c.store.issues[issue.ID] = issue
	c.store.runs[run.ID] = run
	c.store.executions[run.ID] = execution
	return run, execution, nil
}

func (c *Coordinator) Heartbeat(runID, token string) error {
	c.store.mu.Lock()
	defer c.store.mu.Unlock()

	execution, ok := c.store.executions[runID]
	if !ok || execution.Status != LeaseRunning || execution.LeaseToken != token {
		return ErrLeaseLost
	}
	now := c.store.clock.Now().UTC()
	execution.Heartbeat = now
	execution.StaleAfter = now.Add(c.staleWindow)
	c.store.executions[runID] = execution
	return nil
}

func (c *Coordinator) StealStale(runID, runnerID string) (Execution, error) {
	c.store.mu.Lock()
	defer c.store.mu.Unlock()

	execution, ok := c.store.executions[runID]
	if !ok {
		return Execution{}, ErrNotFound
	}
	if execution.Status != LeaseRunning {
		return Execution{}, ErrLeaseLost
	}
	if !c.store.clock.Now().UTC().After(execution.StaleAfter) {
		return Execution{}, ErrLeaseCurrent
	}
	now := c.store.clock.Now().UTC()
	execution.RunnerID = runnerID
	execution.LeaseToken = c.store.token()
	execution.Heartbeat = now
	execution.StaleAfter = now.Add(c.staleWindow)
	c.store.executions[runID] = execution
	return execution, nil
}

func (c *Coordinator) Complete(runID, token, output string) (Outcome, error) {
	c.store.mu.Lock()
	defer c.store.mu.Unlock()

	execution, ok := c.store.executions[runID]
	if !ok || execution.Status != LeaseRunning || execution.LeaseToken != token {
		return Outcome{}, ErrLeaseLost
	}
	run := c.store.runs[runID]
	issue := c.store.issues[run.IssueID]
	run.Status = RunSucceeded
	run.Output = output
	execution.Status = LeaseDone
	issue.Status = IssueInReview
	c.store.runs[runID] = run
	c.store.executions[runID] = execution
	c.store.issues[issue.ID] = issue
	return Outcome{Issue: issue, Run: run, ReviewRequired: true}, nil
}

func (c *Coordinator) Fail(runID, token, rawError string) (Outcome, error) {
	c.store.mu.Lock()
	defer c.store.mu.Unlock()

	execution, ok := c.store.executions[runID]
	if !ok || execution.Status != LeaseRunning || execution.LeaseToken != token {
		return Outcome{}, ErrLeaseLost
	}
	run := c.store.runs[runID]
	issue := c.store.issues[run.IssueID]
	run.Status = RunFailed
	run.Error = rawError
	execution.Status = LeaseFailed
	issue.Status = IssueBlocked
	outcome := Outcome{Issue: issue, Run: run, ReviewRequired: true}
	if classify(rawError) == "provider_network" {
		issue.Status = IssueInProgress
		outcome.Retryable = true
		outcome.ReviewRequired = false
	}
	c.store.runs[runID] = run
	c.store.executions[runID] = execution
	c.store.issues[issue.ID] = issue
	outcome.Issue = issue
	return outcome, nil
}

func classify(raw string) string {
	lower := strings.ToLower(raw)
	switch {
	case strings.Contains(lower, "network"),
		strings.Contains(lower, "connection closed"),
		strings.Contains(lower, "dial tcp"):
		return "provider_network"
	case strings.Contains(lower, "401"),
		strings.Contains(lower, "403"),
		strings.Contains(lower, "permission denied"):
		return "provider_auth"
	default:
		return "business_failure"
	}
}
~~~

Start 依次执行 invocation authorization、Runtime readiness 和 lease claim。任何一步失败都不会创建 Run 或 Execution。

Heartbeat 只接受当前 token。StealStale 模拟 Scheduler v3 的 stale steal，复用同一 Run 并旋转 token。Complete 和 Fail 共享 token 围栏，所以旧 owner 无法覆盖当前 owner。

Fail 只返回 retry/review 决策。retry child 的创建、fire_at 和 workflow handoff 已在 Failure / Review v2 冻结，本次集成不复制那套事务。

## 3. 冻结 M1 验收

创建 M1/reference/delivery/m1_test.go：

~~~go
package delivery

import (
	"errors"
	"testing"
	"time"
)

func fixture() (*Coordinator, *FixedClock) {
	clock := &FixedClock{Current: time.Unix(100, 0).UTC()}
	store := NewStore(clock)
	store.AddRuntime(Runtime{ID: "runtime-1", Status: RuntimeOnline})
	store.AddAgent(Agent{ID: "agent-1", OwnerID: "member-1", RuntimeID: "runtime-1"})
	store.AddIssue(Issue{ID: "issue-1", OwnerID: "member-1", AgentID: "agent-1", Status: IssueTodo})
	return NewCoordinator(store, time.Minute), clock
}

func TestEndToEndSuccessReachesReview(t *testing.T) {
	coordinator, _ := fixture()
	run, execution, err := coordinator.Start("issue-1", "member-1", "runner-a")
	if err != nil {
		t.Fatal(err)
	}
	if err := coordinator.Heartbeat(run.ID, execution.LeaseToken); err != nil {
		t.Fatal(err)
	}
	outcome, err := coordinator.Complete(run.ID, execution.LeaseToken, "changes ready")
	if err != nil {
		t.Fatal(err)
	}
	if outcome.Issue.Status != IssueInReview || !outcome.ReviewRequired ||
		outcome.Run.Status != RunSucceeded {
		t.Fatalf("outcome = %+v", outcome)
	}
}

func TestInvocationAndRuntimeGatesStopBeforeClaim(t *testing.T) {
	coordinator, clock := fixture()
	if _, _, err := coordinator.Start("issue-1", "other-member", "runner-a"); !errors.Is(err, ErrInvocationDenied) {
		t.Fatalf("wrong member error = %v", err)
	}
	clock.Current = clock.Current.Add(time.Second)
	coordinator.store.runtimes["runtime-1"] = Runtime{ID: "runtime-1", Status: RuntimeOffline}
	if _, _, err := coordinator.Start("issue-1", "member-1", "runner-a"); !errors.Is(err, ErrRuntimeWaitable) {
		t.Fatalf("offline runtime error = %v", err)
	}
	issue, _ := coordinator.store.Issue("issue-1")
	if issue.Status != IssueTodo {
		t.Fatalf("issue after rejected starts = %+v", issue)
	}
}

func TestOldLeaseCannotWriteAfterStaleSteal(t *testing.T) {
	coordinator, clock := fixture()
	run, old, err := coordinator.Start("issue-1", "member-1", "runner-a")
	if err != nil {
		t.Fatal(err)
	}
	clock.Current = old.StaleAfter.Add(time.Second)
	current, err := coordinator.StealStale(run.ID, "runner-b")
	if err != nil {
		t.Fatal(err)
	}
	if current.LeaseToken == old.LeaseToken || current.RunnerID != "runner-b" {
		t.Fatalf("current lease = %+v", current)
	}
	if _, err := coordinator.Complete(run.ID, old.LeaseToken, "old"); !errors.Is(err, ErrLeaseLost) {
		t.Fatalf("old completion error = %v", err)
	}
	outcome, err := coordinator.Complete(run.ID, current.LeaseToken, "winner")
	if err != nil || outcome.Issue.Status != IssueInReview {
		t.Fatalf("winner outcome = %+v, err = %v", outcome, err)
	}
}

func TestFailureProjectionSeparatesRetryFromReview(t *testing.T) {
	coordinator, _ := fixture()
	run, execution, err := coordinator.Start("issue-1", "member-1", "runner-a")
	if err != nil {
		t.Fatal(err)
	}
	outcome, err := coordinator.Fail(run.ID, execution.LeaseToken, "connection closed mid-response")
	if err != nil {
		t.Fatal(err)
	}
	if !outcome.Retryable || outcome.ReviewRequired || outcome.Issue.Status != IssueInProgress {
		t.Fatalf("network outcome = %+v", outcome)
	}
}

func TestBusinessFailureProjectsToBlockedReview(t *testing.T) {
	coordinator, _ := fixture()
	run, execution, err := coordinator.Start("issue-1", "member-1", "runner-a")
	if err != nil {
		t.Fatal(err)
	}
	outcome, err := coordinator.Fail(run.ID, execution.LeaseToken, "repository validation failed")
	if err != nil {
		t.Fatal(err)
	}
	if outcome.Retryable || !outcome.ReviewRequired || outcome.Issue.Status != IssueBlocked ||
		outcome.Run.Status != RunFailed {
		t.Fatalf("business failure outcome = %+v", outcome)
	}
}
~~~

在 M1/reference 运行：

~~~bash
go test ./...
go test -race ./...
~~~

预期五个测试通过：

- 正常链路完成 heartbeat、终态写入，并把 Issue 投影为 in_review
- 错误调用者和 offline Runtime 在 claim 前被拒绝
- stale steal 后旧 token 的完成请求返回 ErrLeaseLost
- provider network failure 得到 retryable 决策，Issue 保持 in_progress
- business failure 得到 Review 决策，Issue 进入 blocked

这组测试证明了跨模块交接顺序和状态边界。它没有证明真实 HTTP、PostgreSQL 事务或 Provider 进程已在本地 reference 中运行。

## 4. 对照真实 Multica 检查

源码侧可以用已有测试分别检查组成边界：

~~~bash
cd sources/multica/server
go test ./pkg/taskfailure -run '^(TestClassify|TestNormalizeDaemonReason|TestProviderUnconfigured)' -count=1
go test ./internal/scheduler \
  -run '^(TestConcurrentClaimsSingleWinner|TestStaleStealTerminalUpdateIgnored)' \
  -count=1 -v
go test ./internal/service \
  -run '^(TestWakeupRunKeepsTheRegisteringRunsStep|TestRetryKeepsWorkflowHandoffBrief|TestRuntimeAccessGatesQueuedTaskClaims|TestCompleteTask)' \
  -count=1
~~~

第一条检查失败 taxonomy。第二条检查真实 claim、stale steal 和旧 terminal update。第三条使用 Multica 的 PostgreSQL fixture，检查 wakeup/retry handoff、Runtime access gate 和 `CompleteTask` 的成功、重复回调及迟到回调边界。执行 service 命令前，需要为 `DATABASE_URL` 配置一次性的测试数据库；测试本身会在结束时清理自己的数据。

## M1 冻结条件

M1 完成后，读者应能沿着一条 Run 解释：

~~~text
IssueService.Create
  → CanMemberInvokeAgent
  → AgentReadiness / RuntimeLookup
  → Scheduler claim
  → heartbeat
  → CompleteTaskWithTransition / FailTaskWithTransition
  → Retry or Review decision
  → Issue projection
~~~

成功事实停在 in_review。lease loss、Runtime offline、权限拒绝和 provider network failure 继续保持不同原因。下一步进入具体产品交付能力或下一条产品线的集成门。
