# Orchestration & Heartbeat System

## 1. 개요

Heartbeat는 Paperclip의 **핵심 오케스트레이션 엔진**이다. Agent를 깨우고, 작업을 할당하고, 실행을 관리하고, 결과를 기록하는 전체 실행 루프를 담당한다.

핵심 파일: `server/src/services/heartbeat.ts` (~144KB, 3,950+ lines)

## 2. Heartbeat 트리거 유형

Agent가 깨어나는 방법은 4가지이다:

| 트리거 | 소스 | 설명 |
|--------|------|------|
| `timer` | Cron 스케줄 | Routine의 cron 표현식에 의한 주기적 실행 |
| `assignment` | 작업 할당 | Issue가 Agent에게 할당되면 즉시 깨움 |
| `on_demand` | Board 수동 | Board가 UI에서 수동으로 Agent 호출 |
| `automation` | 시스템 자동 | 콜백, 이벤트, 플러그인에 의한 자동 호출 |

## 3. Wakeup Queue 시스템

Agent를 깨우는 요청은 **큐(Queue)**로 관리된다.

```typescript
AgentWakeupRequest {
  id: UUID
  agentId: UUID
  requestedBy: string     // "board", "system", "agent:{id}"
  reason: string          // "manual", "ping", "callback", "timer"
  status: enum            // pending → fulfilled | cancelled
  idempotencyKey: string  // 중복 요청 병합용
  createdAt: timestamp
}
```

### 큐 처리 흐름

```
enqueueWakeup(agentId, reason):
  1. AgentWakeupRequest 생성 (status: pending)
  2. idempotencyKey로 중복 검사 (같은 키면 기존 요청 재사용)
  3. resumeQueuedRuns() 트리거

resumeQueuedRuns():
  for each pending wakeup request:
    1. Agent가 이미 실행 중인지 확인 (heartbeatRuns status='running')
    2. 최대 동시 실행 수 확인 (기본 1, 최대 10)
    3. withAgentStartLock() 으로 레이스 컨디션 방지
    4. HeartbeatRun 레코드 생성 (status: queued)
    5. WakeupRequest → fulfilled
    6. 실행 시작
```

### 동시성 제어

```
┌─────────────────────────────────────────────┐
│ Per-Agent Start Lock                         │
│ (startLocksByAgent Map)                      │
│                                              │
│  Agent-A: [Lock] ← 동시에 하나만 시작 가능    │
│  Agent-B: [Lock]                             │
│  Agent-C: [Free] ← 시작 가능                 │
└─────────────────────────────────────────────┘
```

- `withAgentStartLock()`: Agent별 뮤텍스. 동일 Agent의 동시 시작 방지.
- `normalizeMaxConcurrentRuns()`: Agent당 최대 동시 실행 수 제한 (기본 1).
- 글로벌 동시 실행 제한은 별도 설정 가능.

## 4. HeartbeatRun 데이터 모델

각 실행은 HeartbeatRun으로 기록된다.

```typescript
HeartbeatRun {
  id: UUID
  companyId: UUID
  agentId: UUID

  // 트리거 정보
  invocationSource: enum    // on_demand | schedule | event | manual
  status: enum              // queued → running → success | failure

  // 실행 결과
  startedAt: timestamp
  finishedAt: timestamp
  exitCode: number | null
  signal: string | null
  error: string | null

  // 사용량
  usageJson: JSONB          // { inputTokens, outputTokens, cachedInputTokens }
  resultJson: JSONB         // 어댑터 반환값

  // 로그 (불변)
  logStore: string          // 로그 저장소 타입
  logRef: string            // 로그 참조 키
  logBytes: number
  logSha256: string         // 무결성 검증
  logCompressed: boolean

  // 세션
  sessionIdBefore: string   // 실행 전 세션
  sessionIdAfter: string    // 실행 후 세션

  // 컨텍스트 스냅샷
  contextSnapshot: JSONB    // 실행 시점의 회사/목표 상태 동결
}
```

## 5. 전체 실행 흐름

```
Phase 1: Pre-Execution (준비)
┌─────────────────────────────────────────────────────┐
│ 1. Wakeup 트리거 수신                                 │
│ 2. HeartbeatRun 생성 (status: queued)                 │
│ 3. 워크스페이스 결정                                    │
│    ├── Project Primary? Task Session? Agent Home?      │
│    └── Git clone 또는 worktree 생성 (필요시)            │
│ 4. 런타임 서비스 확인 (Docker, DB 등)                    │
│ 5. 이전 세션 로드 (sessionCodec.deserialize)            │
│ 6. 예산 확인 (budget check)                            │
│ 7. Agent Instructions 빌드                             │
│ 8. Execution Workspace 생성 (필요시)                    │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
Phase 2: Execution (실행)
┌─────────────────────────────────────────────────────┐
│ 1. HeartbeatRun status → running                      │
│ 2. adapter.execute() 호출                              │
│    ├── 입력: runId, agent, runtime, config, context    │
│    ├── 콜백: onLog(실시간 로그), onMeta, onSpawn        │
│    └── Agent 프로세스 실행 (Claude Code CLI 등)         │
│ 3. Agent가 작업 수행:                                   │
│    ├── Issue 체크아웃 (atomic)                          │
│    ├── 코드 구현, 테스트                                │
│    ├── PR 생성 (WorkProduct)                           │
│    └── Comment 작성 (진행 보고)                         │
│ 4. Agent 프로세스 종료 또는 타임아웃                      │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
Phase 3: Post-Execution (후처리)
┌─────────────────────────────────────────────────────┐
│ 1. AdapterExecutionResult 수신                        │
│ 2. 사용량 기록:                                        │
│    ├── CostEvent 생성 (토큰, 비용)                     │
│    ├── agents.spentMonthlyCents 업데이트               │
│    └── companies.spentMonthlyCents 업데이트            │
│ 3. 예산 정책 평가:                                      │
│    ├── 임계값 도달 → 경고 (BudgetIncident)              │
│    └── hard-stop → 즉시 Agent 일시정지                  │
│ 4. 세션 저장:                                          │
│    ├── sessionCodec.serialize()                        │
│    ├── AgentTaskSession 업데이트                        │
│    └── 세션 컴팩션 체크 (토큰/수명/횟수 임계값)            │
│ 5. 런타임 서비스 상태 persist                            │
│ 6. HeartbeatRun status → success/failure               │
│ 7. Agent status → idle                                 │
└─────────────────────────────────────────────────────┘
```

## 6. Agent에게 전달되는 컨텍스트

Heartbeat가 어댑터를 호출할 때 전달하는 컨텍스트:

```typescript
context = {
  // 워크스페이스 정보
  paperclipWorkspace: {
    cwd: "/project/ai-chat",
    source: "project_primary" | "task_session" | "agent_home",
    projectId: "...",
    workspaceId: "...",
    repoUrl: "https://github.com/...",
    repoRef: "main",
    strategy: "git_worktree" | "project_primary",
    branchName: "feature/login",
    agentHome: "~/.paperclip/agents/{agentId}/"
  },

  // 작업 정보
  issueId: "issue-123",
  taskId: "task-abc",
  wakeReason: "assignment",        // 왜 깨어났는지
  linkedIssueIds: ["issue-123"],   // 연결된 이슈들

  // 회사 컨텍스트
  companyGoal: "AI 챗봇 서비스 런칭",
  projectContext: { ... },

  // 런타임 서비스
  paperclipRuntimeServices: [
    { type: "docker", connectionString: "...", status: "ready" }
  ],

  // 워크스페이스 힌트 (Agent가 활용할 수 있는 추가 정보)
  paperclipWorkspaces: [
    { workspaceId, cwd, repoUrl, repoRef }
  ]
}
```

## 7. 세션 관리 & 컴팩션

### Persistent Sessions

Agent가 작업을 이어서 할 수 있도록 세션 상태를 보존한다.

```typescript
AgentTaskSession {
  companyId: UUID
  agentId: UUID
  adapterType: string
  taskKey: string            // 작업 식별자
  sessionParamsJson: JSONB   // 어댑터별 세션 데이터
  sessionDisplayId: string   // 사람이 읽을 수 있는 세션 ID
  lastRunId: UUID
  lastError: string | null
}
// Unique: (companyId, agentId, adapterType, taskKey)
```

### Session Compaction (세션 컴팩션)

세션이 오래되면 컨텍스트 품질이 저하된다. 자동 컴팩션으로 신선도를 유지한다.

```
컴팩션 트리거 조건 (하나라도 충족 시):
  ├── 토큰 사용량이 임계값 초과
  ├── 세션 생성 후 일정 시간 경과
  └── 실행 횟수가 임계값 초과

컴팩션 동작:
  1. 현재 세션 상태를 요약
  2. 새 세션 생성
  3. 요약된 컨텍스트를 새 세션에 전달
  4. 이전 세션 ID 기록 (감사용)
```

## 8. Routine & Cron 스케줄링

반복적인 작업을 자동화하는 시스템.

### Routine 데이터 모델

```typescript
Routine {
  id: UUID
  companyId: UUID
  assigneeAgentId: UUID
  title: string
  description: string
  priority: enum
  status: enum              // active | paused | completed

  // 동시성 정책
  concurrencyPolicy: enum   // coalesce_if_active | run_parallel
  catchUpPolicy: enum       // skip_missed | run_all
}

RoutineTrigger {
  id: UUID
  routineId: UUID
  kind: enum                // cron | webhook | event
  cronExpression: string    // "0 9 * * 1-5" (평일 9시)
  timezone: string          // "Asia/Seoul"
  nextRunAt: timestamp
  lastFiredAt: timestamp
  secretId: UUID            // webhook 서명용
}

RoutineRun {
  id: UUID
  routineId: UUID
  triggerId: UUID
  status: enum              // received → queued → running → completed | failed
  triggerPayload: JSONB
  linkedIssueId: UUID       // 생성된 Issue
  idempotencyKey: string    // 중복 실행 방지
}
```

### Cron 처리 흐름

```
tickTimers() (주기적으로 실행):
  for each active routine:
    1. nextCronTickInTimeZone() 으로 다음 실행 시간 계산
    2. 현재 시간이 실행 시간을 지났는지 확인
    3. 동시성 정책 체크:
       ├── coalesce_if_active: 이미 실행 중이면 스킵 (status: "skipped")
       └── run_parallel: 새 실행 생성
    4. RoutineRun 생성
    5. Issue 생성 (작업 내용으로)
    6. Agent에게 할당 → Wakeup 트리거
```

### Cron 표현식

표준 5-필드 Unix cron 형식. 사용자 로컬 타임존으로 평가된다.

```
분(0-59) 시(0-23) 일(1-31) 월(1-12) 요일(0-6, 일=0)

예시:
  "0 9 * * *"     → 매일 오전 9시
  "0 9 * * 1-5"   → 평일 오전 9시
  "30 8 * * 1"    → 매주 월요일 오전 8:30
  "0 0 1 * *"     → 매월 1일 자정
  "*/15 * * * *"  → 15분마다
```

## 9. 전체 자율 운영 루프

```
┌──────────────────────────────────────────────────────────┐
│ Step 1: Board가 회사 설정                                  │
│ → Goal 정의, Agent 채용, 예산 설정, 프로젝트 생성            │
└──────────────────────────────┬───────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────┐
│ Step 2: Agent 깨우기 (Heartbeat Trigger)                   │
│ → Cron 스케줄, Issue 할당, Board 수동, Webhook              │
└──────────────────────────────┬───────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────┐
│ Step 3: Adapter가 Agent 호출                               │
│ → Claude Code CLI, Codex, HTTP Webhook 등                  │
│ → Agent에게 작업 컨텍스트 전달 (Issue, Workspace, Goal)      │
└──────────────────────────────┬───────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────┐
│ Step 4: Agent 실행                                         │
│ → Atomic Checkout → 코드 구현 → PR 생성 → Comment 보고      │
└──────────────────────────────┬───────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────┐
│ Step 5: 결과 기록                                          │
│ → HeartbeatRun (성공/실패, 로그)                            │
│ → CostEvent (토큰, 비용)                                   │
│ → AgentTaskSession (세션 저장)                              │
│ → ActivityLog (감사)                                       │
└──────────────────────────────┬───────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────┐
│ Step 6: Board 리뷰                                         │
│ → Issue 진행 확인, WorkProduct 리뷰, 비용 확인               │
│ → 승인 또는 피드백 Comment 작성                              │
└──────────────────────────────┬───────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────┐
│ Step 7: 다음 Heartbeat에서 피드백 반영 → 반복                │
└──────────────────────────────────────────────────────────┘
```

## 10. 설계 시사점 & 한계

### 강점
- **Heartbeat 패턴**: 중앙 집중 오케스트레이션으로 안정적 실행 관리
- **세션 지속성**: Agent가 매번 처음부터 시작하지 않아도 됨
- **Cron + Webhook + Event**: 다양한 트리거 방식 지원
- **Idempotent Wakeup**: 중복 깨우기 자동 병합
- **Context Snapshot**: 실행 시점 컨텍스트 동결로 재현 가능

### 개선 가능 포인트
- **Polling 기반 Heartbeat**: Agent가 능동적으로 작업을 찾지 않고, 시스템이 주기적으로 깨움. 즉각적인 반응이 어려움.
- **단일 Agent 동시성 기본 1**: 하나의 Agent가 여러 작업을 병렬 실행하기 어려움. 작업량이 많은 Agent에게 병목.
- **피드백 루프 지연**: Board 리뷰 → Comment → 다음 Heartbeat까지 대기 → Agent 반영. 간단한 수정도 여러 Heartbeat 사이클이 필요할 수 있음.
- **Heartbeat 크기**: heartbeat.ts가 ~144KB, 3,950+ lines. 단일 파일에 너무 많은 로직이 집중되어 있어 유지보수 부담.
- **Agent 간 직접 호출 없음**: Agent-A가 Agent-B에게 직접 작업을 요청할 수 없음. 반드시 Issue 생성 → 할당 → Heartbeat 사이클을 거쳐야 함.
- **실시간 협업 불가**: 동일 Issue에서 여러 Agent가 동시에 작업하는 패턴 미지원.
