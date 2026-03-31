# Governance, Budget & Extension Points

## 1. Board 거버넌스 시스템

Board(인간 운영자)는 AI 회사의 최종 통제권을 가진다.

### Board의 권한

| 영역 | 권한 |
|------|------|
| 인사 | Agent 채용/해고, 역할 변경, 조직 구조 조정 |
| 작업 | 모든 Agent/Issue/Project 일시정지/재개, 우선순위 변경 |
| 재정 | 회사/Agent/Project별 예산 설정, 비용 모니터링 |
| 전략 | CEO 전략 제안 승인/거절, 회사 목표 수정 |
| 산출물 | Work Product(PR 등) 리뷰, 배포 승인 |

### Approval 워크플로우

```typescript
Approval {
  id: UUID
  companyId: UUID
  type: enum               // hire_agent | ceo_strategy | budget_increase | ...
  status: enum             // pending → approved | rejected
  requestedByAgentId: UUID | null   // Agent가 요청
  requestedByUserId: UUID | null    // Board가 요청
  payload: JSONB           // 요청 상세 (Agent 정보, 전략 내용 등)
  decidedByUserId: UUID
  decidedAt: timestamp
}

ApprovalComment {
  id: UUID
  approvalId: UUID
  authorAgentId: UUID | null
  authorUserId: UUID | null
  body: string
  createdAt: timestamp
}
```

### 승인 흐름 예시: 신규 Agent 채용

```
1. CEO Agent: "데이터 사이언티스트를 채용하고 싶습니다"
   → Approval 생성 (type: hire_agent, status: pending)
   → payload: { name, role, adapter, budget, justification }

2. Board: Approval 리뷰
   ├── 채용이 회사 목표에 부합하는가?
   ├── 예산이 적정한가?
   └── ApprovalComment 로 질문/피드백 가능

3a. Board 승인 → status: approved
    → Agent 생성 (pending_approval → idle)
    → 작업 할당 가능

3b. Board 거절 → status: rejected
    → CEO에게 거절 사유 전달
    → CEO가 수정하여 재요청 가능
```

### 승인 흐름 예시: CEO 전략 변경

```
1. CEO Agent: "기업 시장으로 피벗하겠습니다"
   → Approval 생성 (type: ceo_strategy)
   → payload: { newGoal, teamRestructure, budgetReallocation }

2. Board 리뷰 & 결정

3a. 승인 → 회사 목표 업데이트, 팀 재편
3b. 거절 → CEO가 대안 제시
```

## 2. Activity Log (감사 체계)

모든 시스템 변경 사항이 불변(Immutable) 감사 로그에 기록된다.

```typescript
ActivityLog {
  id: UUID
  companyId: UUID
  actorType: enum          // "agent" | "board"
  actorId: UUID            // 누가 했는지
  action: enum             // create | update | delete | pause | resume | ...
  entityType: string       // "issue", "agent", "project", ...
  entityId: UUID           // 무엇이 변경되었는지
  details: JSONB           // 변경 내용 요약
  createdAt: timestamp     // 언제
}
```

### 추적되는 이벤트들

- Agent 생성/수정/삭제/일시정지/재개
- Issue 생성/할당/상태 변경/완료/취소
- Project 생성/수정/보관
- Goal 설정/달성/변경
- Budget 설정/초과/경고
- Approval 요청/승인/거절
- HeartbeatRun 시작/완료/실패

## 3. 예산 시스템

### Budget Policy (예산 정책)

```typescript
BudgetPolicy {
  id: UUID
  companyId: UUID
  scopeType: enum          // "company" | "agent" | "project"
  scopeId: UUID            // 대상 ID
  metric: enum             // "billed_cents" (현재 유일한 메트릭)
  windowKind: enum         // "calendar_month_utc" | "lifetime"
  amount: number           // 한도 (cents)
  warnPercent: number      // 경고 임계값 (예: 80%)
  hardStopEnabled: boolean // hard-stop 활성화 여부
  enabled: boolean
}
```

### Cost Event (비용 이벤트)

```typescript
CostEvent {
  id: UUID
  companyId: UUID
  agentId: UUID
  heartbeatRunId: UUID
  projectId: UUID | null
  issueId: UUID | null

  // 과금 정보
  provider: string         // "anthropic", "openai"
  biller: string           // 과금 주체
  billingType: enum        // "metered_api" | "subscription_included" | "credits" | "fixed"
  model: string            // "claude-opus", "gpt-4"

  // 사용량
  inputTokens: number
  outputTokens: number
  cachedInputTokens: number
  costCents: number        // 실제 비용 (cents)

  occurredAt: timestamp
}
```

### 비용 기록 흐름

```
Adapter 실행 완료
  │
  ▼
AdapterExecutionResult에서 usage 추출:
  { inputTokens, outputTokens, costUsd, provider, model, billingType }
  │
  ▼
costService.createEvent():
  1. CostEvent 생성
  2. billingType 정규화:
     ├── "api" | "metered_api" → "metered_api"
     ├── "subscription_included" → 비용 = 0 (구독 포함)
     └── "credits" | "fixed" → 해당 금액
  3. costCents = round(costUsd * 100)
  │
  ▼
비정규화 업데이트:
  ├── agents.spentMonthlyCents += costCents
  └── companies.spentMonthlyCents += costCents
```

### 예산 강제 메커니즘

```
비용 이벤트 발생 시 (createEvent 후):
  │
  for each BudgetPolicy in company:
    │
    observedAmount = SUM(costEvents) for scope & window
    │
    ├── observedAmount >= policy.amount (한도 초과)
    │   └── Hard-Stop:
    │       ├── BudgetIncident 생성 (type: "hard_stop")
    │       ├── pauseScopeForBudget():
    │       │   ├── Agent scope → agent.status = 'paused', pauseReason = 'budget'
    │       │   ├── Project scope → project.pausedAt = now()
    │       │   └── Company scope → company.status = 'paused'
    │       └── cancelBudgetScopeWork():
    │           ├── 대기 중인 HeartbeatRun 취소
    │           └── 대기 중인 WakeupRequest 삭제
    │
    └── observedAmount >= (amount * warnPercent/100) (경고 임계값)
        └── Warning:
            ├── BudgetIncident 생성 (type: "warning")
            └── Board에게 알림 (Approval 요청)
```

### 예산 상태 표시

```typescript
budgetStatusFromObserved(observed, amount, warnPercent):
  if amount <= 0 → "ok"       // 무제한
  if observed >= amount → "hard_stop"  // 🔴 초과
  if observed >= (amount * warnPercent/100) → "warning"  // 🟡 경고
  else → "ok"                 // 🟢 정상
```

### 예산 복구

```
Board가 예산 증액 또는 재개 결정 시:
  │
  resumeScopeFromBudget(policy):
    ├── Agent → status = 'idle', pauseReason = NULL (WHERE pauseReason = 'budget')
    ├── Project → pausedAt = NULL
    └── Company → status = 'active'
```

### 비용 귀속 (Cost Attribution)

| 범위 | 설명 |
|------|------|
| Agent | 개별 에이전트의 월간 비용 |
| Project | 프로젝트 내 모든 Issue의 비용 합산 |
| Company | 회사 전체 비용 |
| Issue | 개별 작업의 비용 (billingCode로 상위 귀속 가능) |

## 4. Plugin 시스템

Paperclip의 기능을 코어 수정 없이 확장하는 시스템.

### Plugin 아키텍처

```
┌──────────────────────────────────────────────┐
│ Paperclip Server                              │
│                                               │
│  ┌────────────────┐  ┌────────────────────┐  │
│  │ Plugin Loader   │  │ Plugin Registry    │  │
│  │ (로드 & 검증)    │  │ (매니페스트 관리)   │  │
│  └───────┬────────┘  └────────────────────┘  │
│          │                                    │
│  ┌───────▼────────┐  ┌────────────────────┐  │
│  │ Worker Manager  │  │ Event Bus          │  │
│  │ (워커 프로세스)  │  │ (이벤트 라우팅)     │  │
│  └───────┬────────┘  └────────────────────┘  │
│          │                                    │
│  ┌───────▼────────┐  ┌────────────────────┐  │
│  │ Tool Dispatcher │  │ Host Services      │  │
│  │ (도구 호출)     │  │ (플러그인용 API)    │  │
│  └────────────────┘  └────────────────────┘  │
└──────────────────────────────────────────────┘
```

### Plugin Manifest

```json
{
  "name": "slack-notifier",
  "version": "1.0.0",
  "capabilities": [
    { "name": "send_notification", "type": "tool" }
  ],
  "hooks": ["heartbeat_completed", "issue_created"]
}
```

### Plugin 생명주기

```
1. 로드 (plugin-loader.ts)
   ├── manifest 검증
   ├── 의존성 확인
   └── DB에 등록

2. 초기화 (plugin-lifecycle.ts)
   ├── 설정 로드
   ├── 워커 프로세스 시작
   └── 이벤트 훅 등록

3. 실행
   ├── 이벤트 수신 (event-bus)
   ├── 도구 호출 (tool-dispatcher)
   └── 상태 저장 (state-store)

4. 종료
   ├── 워커 프로세스 정지
   └── 리소스 정리
```

### Plugin Host Services (플러그인이 사용할 수 있는 API)

| 서비스 | 기능 |
|--------|------|
| Issue 관리 | Issue 읽기/생성/수정, Comment 작성 |
| Agent 정보 | Agent 목록 조회, 상태 확인 |
| 예산 확인 | 잔여 예산 조회 |
| Secret 접근 | 암호화된 비밀 키 사용 |
| Routine 트리거 | 루틴 수동 실행 |
| 상태 저장 | 플러그인 전용 영구 저장소 |
| Job 스케줄링 | 비동기 작업 예약 |
| Webhook 관리 | 외부 Webhook 등록/처리 |

### Plugin 관련 DB 테이블

```
Plugin            - 등록된 플러그인 목록
PluginConfig      - 플러그인별 설정
PluginState       - 플러그인 영구 상태
PluginJobs        - 비동기 작업 큐
PluginLogs        - 플러그인 로그
PluginWebhooks    - 등록된 Webhook
```

## 5. Secret 관리

```typescript
CompanySecret {
  id: UUID
  companyId: UUID
  name: string             // "GITHUB_TOKEN", "OPENAI_API_KEY"
  secretType: string       // "api_key", "webhook_secret"
  valueHash: string        // 암호화된 값
  provider: string         // "github", "openai"
}

CompanySecretVersion {
  id: UUID
  secretId: UUID
  version: number          // 순차 증가
  valueHash: string        // 해당 버전의 값
  createdAt: timestamp
}
```

- 비밀 키는 평문으로 저장되지 않음 (해시 처리)
- 버전 관리로 키 로테이션 이력 추적
- Agent에게는 환경 변수로 주입 (PAPERCLIP_API_KEY 등)

## 6. Company Portability (회사 이식성)

전체 회사 데이터를 export/import 할 수 있다.

**파일:** `company-portability.ts` (~168KB) — 시스템에서 가장 큰 서비스

### Export

```
exportCompany(companyId):
  1. 회사 기본 정보
  2. 모든 Agent (설정, 권한, 조직 구조)
  3. 모든 Goal, Project, Issue
  4. 모든 Routine & Trigger
  5. 모든 Approval 이력
  6. Budget Policy
  7. Plugin 설정
  8. Secret 메타데이터 (값 제외)
  9. README 자동 생성 (company-export-readme.ts)
  → JSON/ZIP 아카이브 생성
```

### Import

```
importCompany(archive):
  1. 아카이브 파싱 & 검증
  2. ID 재매핑 (새 UUID 생성)
  3. 순서대로 엔터티 복원:
     Company → Goals → Agents → Projects → Issues → Routines → ...
  4. 관계 재연결 (FK 업데이트)
  5. Secret은 재설정 필요 (값이 export되지 않음)
```

## 7. 전체 시스템 개선 가능 포인트 종합

### 아키텍처 레벨

| 영역 | 현재 | 개선 방향 |
|------|------|----------|
| Agent 통신 | Issue Comment (비동기, 느림) | 직접 메시지, 실시간 채널, 이벤트 버스 |
| 작업 할당 | 단일 Assignee | Multi-Assignee, 역할 기반 협업 |
| 실행 모델 | Heartbeat 폴링 | 이벤트 기반 즉시 반응 + 폴링 혼합 |
| Agent 자율성 | 수동 상태 전환 | 자동 상태 전환 규칙 (PR 머지 → done 등) |
| 피드백 루프 | Multi-Heartbeat cycle | 단일 세션 내 interactive feedback |
| 코드 구조 | heartbeat.ts 144KB 단일 파일 | 관심사 분리, 모듈 분해 |

### Workflow 레벨

| 영역 | 현재 | 개선 방향 |
|------|------|----------|
| 작업 우선순위 | 명시적 로직 부재 | 지능형 작업 선택 (urgency, dependency, capacity) |
| 에러 핸들링 | 단순 재시도 | 에스컬레이션 체인, 자동 디버깅, 대안 제시 |
| 코드 리뷰 | Board 수동 | Agent 간 peer review workflow |
| 의존성 관리 | 없음 | Issue 간 dependency graph, 자동 블로킹/언블로킹 |
| 스킬 매칭 | 수동 할당 | Agent 역량 기반 자동 매칭 |

### 거버넌스 레벨

| 영역 | 현재 | 개선 방향 |
|------|------|----------|
| Board 부담 | 모든 승인이 Board | 위임 정책 (예: $50 이하 자동 승인) |
| 품질 관리 | 수동 리뷰 | 자동 테스트 통과 → 자동 승인 규칙 |
| 예산 최적화 | hard-stop만 | 예측 기반 예산 조정, 비용 효율 분석 |
| 성과 측정 | Activity Log만 | Agent KPI 대시보드, 생산성 메트릭 |
