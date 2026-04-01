# Paperclip Agent 활용 전략 심층 분석

---

## Part 1: Agent에게 주입되는 프롬프트 구조

Agent가 깨어날 때 실제로 "보는" 것은 3개 레이어의 조합이다.

### 1.1 프롬프트 3-Layer 구조

```
┌─────────────────────────────────────────────────────────┐
│  Layer 1: System Prompt (--append-system-prompt-file)    │
│  ═══════════════════════════════════════════════════════  │
│  • Agent Instructions Bundle (AGENTS.md 기반)            │
│  • 에이전트별 역할, 행동 규칙, 도메인 지식                  │
│  • 경로 디렉티브: "이 지시는 {path}에서 로드됨"            │
│                                                          │
│  → Claude CLI의 system prompt에 추가됨                    │
│  → 매 실행마다 동일하게 적용                               │
└─────────────────────────────────────────────────────────┘
                         +
┌─────────────────────────────────────────────────────────┐
│  Layer 2: Skills (--add-dir skillsDir)                   │
│  ═══════════════════════════════════════════════════════  │
│  • paperclip/ → 핵심 스킬 (API 레퍼런스, 절차, 규칙)      │
│  • paperclip-create-agent/ → Agent 생성 스킬             │
│  • paperclip-create-plugin/ → Plugin 생성 스킬           │
│  • para-memory-files/ → 메모리/참조 파일 관리 스킬         │
│                                                          │
│  → Claude Code의 스킬 시스템으로 마운트                    │
│  → Agent가 필요시 참조 가능                               │
└─────────────────────────────────────────────────────────┘
                         +
┌─────────────────────────────────────────────────────────┐
│  Layer 3: Stdin Prompt (3-Section Composition)           │
│  ═══════════════════════════════════════════════════════  │
│  Section A: Bootstrap Prompt (첫 세션에서만)              │
│  Section B: Session Handoff (세션 회전 시)                │
│  Section C: Heartbeat Prompt (매 실행마다)                │
│                                                          │
│  → Claude CLI의 stdin으로 전달                            │
│  → 템플릿 변수 치환 후 전달                               │
└─────────────────────────────────────────────────────────┘
```

### 1.2 각 Layer의 상세 내용

**Layer 1 - Agent Instructions Bundle:**

```
저장 위치: companies/{companyId}/agents/{agentId}/instructions/
엔트리 파일: AGENTS.md (기본값, instructionsEntryFile로 변경 가능)
모드: "managed" (Paperclip 내부) 또는 "external" (외부 경로)

파일 스캔 규칙:
  - 재귀적으로 모든 파일 탐색
  - 제외: .git, node_modules, .DS_Store, __pycache__, .pyc, .pyo
  - 정렬된 상대 경로 목록 반환
  - 언어는 확장자로 자동 감지

주입 방식:
  1. AGENTS.md 파일 읽기
  2. 경로 디렉티브 추가: "이 지시는 {path}에서 로드됨. 상대 경로는 {dir}/을 기준으로 해석"
  3. 임시 파일에 합쳐서 --append-system-prompt-file로 전달
```

**Layer 2 - Skills (핵심 스킬 `paperclip/SKILL.md`의 내용):**

Agent가 깨어나면 실행하는 **Heartbeat Procedure 9단계**:

```
1. GET /api/agents/me          → 자기 정보 확인 (신원 확인)
2. 승인 요청 처리                → PAPERCLIP_APPROVAL_ID가 있으면 후속 처리
3. GET /api/agents/me/inbox-lite → 할당된 작업 목록 조회
4. 작업 선택                     → PAPERCLIP_TASK_ID 우선, 없으면 in_progress > todo 순
5. POST /api/issues/{id}/checkout → 작업 잠금 (Atomic Checkout)
6. GET /api/issues/{id}/heartbeat-context → 작업 컨텍스트 로드
7. 실제 작업 수행               → 코드 작성, 테스트, PR 생성 등
8. 상태 업데이트 & 커뮤니케이션  → Comment 작성, 상태 변경
9. 위임 필요시 하위 작업 생성    → 새 Issue 생성 & 다른 Agent에 할당
```

**Layer 3 - Stdin Prompt 구성:**

```typescript
// 3개 섹션을 순서대로 합침
const prompt = joinPromptSections([
  renderedBootstrapPrompt,      // Section A: 첫 세션에서만 (세션 초기화 지시)
  sessionHandoffNote,           // Section B: 세션 회전 시 (이전 컨텍스트 요약)
  renderedPrompt,               // Section C: 매번 (현재 작업 지시)
]);

// 템플릿 변수:
{agent.id}        → 에이전트 UUID
{agent.name}      → 에이전트 이름
{agent.companyId} → 회사 ID
{runId}           → 현재 실행 ID
{context.*}       → 런타임 컨텍스트 전체
```

### 1.3 Agent가 탐색할 수 있는 문서 & 전략

Agent가 깨어난 후 추가 정보를 얻는 경로는 다음과 같다:

```
┌──────────────────────────────────────────────────────────────┐
│  즉시 접근 가능 (프롬프트에 포함)                               │
│  ├── Agent Instructions (AGENTS.md)                           │
│  ├── Skills (SKILL.md 파일들)                                 │
│  ├── 환경 변수 (PAPERCLIP_TASK_ID, WORKSPACE_CWD 등)          │
│  └── Stdin Prompt (작업 지시, 세션 핸드오프)                    │
├──────────────────────────────────────────────────────────────┤
│  API 호출로 탐색 (Heartbeat Procedure 중)                      │
│  ├── GET /agents/me → 자기 역할, 권한, 설정                    │
│  ├── GET /agents/me/inbox-lite → 할당된 작업 목록               │
│  ├── GET /issues/{id}/heartbeat-context → 작업 상세 컨텍스트    │
│  │   └── 이슈 설명, 댓글 이력, 부모 이슈, 프로젝트 목표          │
│  ├── GET /issues/{id}/comments → 이전 대화 이력                │
│  └── GET /issues/{id}/work-products → 기존 산출물 (PR 등)      │
├──────────────────────────────────────────────────────────────┤
│  파일시스템 탐색 (Execution Workspace 내)                       │
│  ├── Workspace CWD 내 소스 코드                                │
│  ├── README.md, CONTRIBUTING.md 등 프로젝트 문서                │
│  ├── .claude/ 디렉토리 내 추가 스킬/설정                        │
│  └── Git 이력 (git log, git diff 등)                           │
└──────────────────────────────────────────────────────────────┘
```

**핵심 설계 의도:** Agent는 "필요한 것만 점진적으로 로드"하는 전략을 취한다. 전체 코드베이스를 한번에 읽지 않고, Issue의 heartbeat-context API를 통해 해당 작업에 필요한 컨텍스트만 가져온다.

---

## Part 2: 에이전트 간 통신 프로토콜

### 2.1 현재 통신 아키텍처

Paperclip에서 Agent 간 직접 통신은 **존재하지 않는다**. 모든 통신은 Paperclip 서버를 중개자로 하는 비동기 메시지 전달이다.

```
Agent A                    Paperclip Server                    Agent B
  │                              │                                │
  │ POST /issues/{id}/comments   │                                │
  │ body: "PR #42 완료, 리뷰 부탁"│                                │
  │ ────────────────────────────→ │                                │
  │                              │  DB: issue_comments INSERT      │
  │                              │  DB: issues.updatedAt UPDATE    │
  │                              │                                │
  │                              │  (다음 Heartbeat에서)            │
  │                              │  Wakeup Agent B                │
  │                              │ ──────────────────────────────→ │
  │                              │                                │
  │                              │  GET /issues/{id}/comments      │
  │                              │ ←────────────────────────────── │
  │                              │                                │
  │                              │  "PR #42 완료, 리뷰 부탁" 확인   │
```

### 2.2 통신 채널 유형

| 채널 | 메커니즘 | 지연 시간 | 용도 |
|------|---------|----------|------|
| **Issue Comment** | POST /issues/{id}/comments | 다음 Heartbeat (수초~수분) | 진행 보고, 질문, 피드백 |
| **Issue 할당** | PATCH /issues/{id} + assigneeAgentId | 즉시 Wakeup 가능 | 작업 위임 |
| **하위 Issue 생성** | POST /issues + parentId | 다음 Heartbeat | 작업 분할 & 위임 |
| **Approval 요청** | POST /approvals | Board 응답 대기 | 권한 필요 작업 |
| **Work Product** | POST /issues/{id}/work-products | 비동기 | PR, 배포 링크 공유 |

### 2.3 통신 프로토콜의 실질적 규칙

```
규칙 1: 모든 통신은 Issue에 바인딩된다
  → Agent 간 "잡담"은 불가. 반드시 Issue 컨텍스트 안에서만 소통.

규칙 2: Comment는 마크다운 텍스트
  → 구조화된 메시지 포맷 없음. 자연어 기반.
  → Agent가 파싱해야 할 구조화 데이터는 환경 변수나 API로 전달.

규칙 3: Wakeup은 단방향
  → Agent A가 Agent B를 직접 깨울 수 없음.
  → Issue 할당 시 시스템이 자동으로 Agent B를 깨움.
  → 또는 Agent B의 다음 스케줄된 Heartbeat에서 Comment 발견.

규칙 4: 409 Conflict = 양보
  → 두 Agent가 같은 Issue를 동시에 checkout 시도하면 늦은 쪽이 409.
  → 409를 받은 Agent는 재시도하지 않고 다른 작업으로 이동.

규칙 5: X-Paperclip-Run-Id 헤더 필수
  → 모든 mutating API 호출에 현재 Run ID를 포함해야 함.
  → 감사 추적 및 비용 귀속을 위해.
```

### 2.4 Agent가 사용 가능한 API 엔드포인트

| 카테고리 | 엔드포인트 | 용도 |
|---------|-----------|------|
| **자기 정보** | `GET /agents/me` | 역할, 권한, 설정 확인 |
| | `GET /agents/me/inbox-lite` | 할당된 작업 경량 목록 |
| | `GET /agents/me/inbox/mine` | 전체 받은 편지함 |
| **Issue 관리** | `POST /companies/{cid}/issues` | 새 Issue 생성 |
| | `PATCH /issues/{id}` | 상태/할당 변경 |
| | `POST /issues/{id}/checkout` | 작업 잠금 |
| | `POST /issues/{id}/release` | 잠금 해제 |
| | `POST /issues/{id}/comments` | 댓글 작성 |
| | `GET /issues/{id}/heartbeat-context` | 작업 컨텍스트 |
| **승인** | `GET /approvals` | 승인 상태 확인 |
| | `POST /approvals` | 승인 요청 |
| **Heartbeat** | `POST /agents/{id}/heartbeat/invoke` | 자기 자신 재호출 (본인만) |

**인증 방식:**
- JWT 토큰: `PAPERCLIP_API_KEY` 환경 변수로 주입
- `createLocalAgentJwt()` 로 단기 토큰 생성
- Agent ID 기반 접근 제어 (자기 소속 회사 리소스만)

---

## Part 3: 에이전트 생성 & 평가

### 3.1 에이전트 생성 경로

에이전트는 두 가지 경로로 생성된다:

```
경로 1: Board가 직접 생성 (UI)
  Board → UI → POST /agents → create() → pending_approval → Board 승인 → idle

경로 2: CEO Agent가 채용 요청
  CEO Agent → POST /approvals (type: hire_agent) → Board 리뷰
    → 승인 → Agent 자동 생성 → pending_approval → Board 재승인 → idle
    → 거절 → CEO에게 거절 사유 전달
```

**생성 시 필요한 정보:**

```typescript
{
  name: "데이터 분석가",          // 표시 이름
  role: "analyst",               // 역할 코드
  title: "Senior Data Analyst",  // 직함
  reportsTo: "ceo-agent-id",     // 매니저 (조직 트리)
  adapterType: "claude_local",   // 실행 어댑터
  adapterConfig: {
    model: "claude-sonnet-4-5-20250514",
    effort: "high",
    maxTurns: 50,
    chrome: false,
    dangerouslySkipPermissions: true
  },
  runtimeConfig: {
    command: "claude",
    timeoutSec: 600,
    heartbeat: {
      intervalSec: 300,
      sessionCompaction: {
        enabled: true,
        maxSessionRuns: 100,
        maxRawInputTokens: 1000000,
        maxSessionAgeHours: 48
      }
    }
  },
  budgetMonthlyCents: 500000,    // $5,000/월
  permissions: { ... }
}
```

### 3.2 현재 평가 체계

**Promptfoo 기반 Phase 0 Bootstrap 평가 (evals/):**

```yaml
# 8개 테스트 케이스, 4개 모델에서 교차 테스트

Core Behaviors (6개):
  1. Assignment Pickup     → in_progress 작업을 todo보다 먼저 선택하는가?
  2. Progress Update       → 종료 전에 진행 상황 Comment를 작성하는가?
  3. Blocked Reporting     → 차단된 작업에 사유를 설명하는가?
  4. No Work Exit          → 할당된 작업이 없을 때 깨끗하게 종료하는가?
  5. Checkout Before Work  → 작업 전에 반드시 checkout하는가?
  6. Conflict Handling     → 409 응답을 받으면 재시도 않고 다른 작업으로 가는가?

Governance (2개):
  7. Approval Required     → 권한 필요 작업에 승인 요청을 하는가?
  8. Company Boundary      → 타 회사 리소스 접근을 거부하는가?

테스트 대상 모델:
  - Claude Sonnet
  - GPT-4.1
  - Codex-5.4
  - Gemini-2.5-pro
```

**현재 추적되는 메트릭:**

| 메트릭 | 위치 | 내용 |
|--------|------|------|
| Token Usage | `agentRuntimeState` | 누적 input/output/cached tokens |
| Cost | `costEvents` + `agentRuntimeState.totalCostCents` | 누적 비용(cents) |
| Run Status | `heartbeatRuns.status` | success/failure 비율 |
| Last Error | `agentRuntimeState.lastError` | 최근 오류 |
| Exit Code | `heartbeatRuns.exitCode` | 프로세스 종료 코드 |

**추적되지 않는 것 (개선 기회):**

```
❌ 작업 완료율 (할당 대비 완료)
❌ 평균 작업 소요 시간
❌ 코드 품질 점수
❌ PR 승인률 / 리젝트률
❌ Agent 간 재작업 빈도
❌ 1회 Heartbeat당 생산성 (토큰 대비 산출물)
❌ 에스컬레이션 빈도
❌ 자율 의사결정 정확도
```

### 3.3 Config Revision (설정 버전 관리)

Agent 설정 변경은 전수 이력 관리된다:

```typescript
AgentConfigRevision {
  id: UUID
  agentId: UUID
  source: "patch" | "rollback"
  changedKeys: ["model", "maxTurns"]        // 변경된 키 목록
  beforeConfig: { model: "opus", ... }       // 변경 전 전체 스냅샷
  afterConfig: { model: "sonnet", ... }      // 변경 후 전체 스냅샷
  createdByAgentId / createdByUserId         // 누가 변경했는지
  rolledBackFromRevisionId                   // 롤백인 경우 원본
}
```

- `GET /agents/{id}/config-revisions` → 변경 이력 조회
- `POST /agents/{id}/config-revisions/{revId}/rollback` → 특정 시점으로 롤백

---

## Part 4: 모델 변경과 성능 차이 극복

### 4.1 현재 모델 설정 방식

```typescript
// packages/adapters/claude-local/src/server/execute.ts
const model = asString(config.model, "");  // adapterConfig.model에서 읽음

if (model) args.push("--model", model);    // Claude CLI에 전달
```

- 모델은 Agent별 `adapterConfig.model`로 고정 설정
- 런타임에 동적 변경 없음
- 폴백 로직 없음 (Claude CLI 내부에 위임)

**Billing Type 자동 감지:**
```typescript
function resolveClaudeBillingType(env): "api" | "subscription" {
  return hasNonEmptyEnvValue(env, "ANTHROPIC_API_KEY") ? "api" : "subscription";
}
```

**모델 탐색 API:**
```
GET /companies/{cid}/adapters/{type}/models        → 사용 가능 모델 목록
GET /companies/{cid}/adapters/{type}/detect-model   → 현재 감지된 모델
```

### 4.2 모델 변경 시 발생하는 문제

```
문제 1: 프롬프트 호환성
  → Opus용으로 최적화된 프롬프트가 Sonnet에서 다르게 동작
  → 특히 복잡한 추론, 코드 생성, 도구 사용 패턴에서 차이

문제 2: 세션 연속성 단절
  → 모델 변경 시 이전 세션 컨텍스트가 새 모델에 최적화되지 않음
  → 세션 컴팩션의 핸드오프 품질이 모델에 따라 다름

문제 3: 비용-성능 트레이드오프
  → Opus: 높은 품질, 높은 비용 → 예산 빨리 소진
  → Haiku: 낮은 비용, 제한된 능력 → 복잡한 작업 실패 가능

문제 4: 도구 사용 패턴 차이
  → 모델마다 API 호출 패턴, 에러 처리 방식이 다름
  → Checkout→작업→Comment 순서를 모든 모델이 정확히 따르지 않을 수 있음
```

### 4.3 현재 시스템이 제공하는 극복 수단

```
1. Promptfoo 교차 테스트
   → 4개 모델(Claude, GPT-4.1, Codex, Gemini)에서 동일 테스트 실행
   → 모델 변경 전 호환성 검증 가능

2. Config Revision & Rollback
   → 모델 변경 후 성능 저하 시 즉시 이전 설정으로 롤백
   → changedKeys로 정확히 "model"이 바뀐 시점 추적

3. Session Compaction Policy 조정
   → 모델별로 다른 컴팩션 정책 적용 가능
   → 약한 모델: maxSessionRuns 줄여서 더 자주 세션 초기화
   → 강한 모델: maxRawInputTokens 늘려서 긴 컨텍스트 활용

4. Effort 파라미터
   → claude --effort high/medium/low
   → 작업 복잡도에 따라 thinking effort 조절
   → 간단한 작업에는 낮은 effort + 빠른 모델

5. 예산 정책 분리
   → Agent별 독립 예산 → 비싼 모델을 쓰는 Agent만 별도 관리
   → warnPercent로 조기 경고
```

### 4.4 추가로 구현 가능한 전략

```
전략 1: 작업 복잡도 기반 모델 라우팅
  ┌────────────────────────────────────────────────┐
  │  Issue Priority/Complexity → Model Selection    │
  │                                                 │
  │  critical + 코드 아키텍처 → Opus (최고 품질)      │
  │  high + 코드 구현       → Sonnet (균형)          │
  │  medium + 단순 수정     → Haiku (비용 효율)       │
  │  low + 문서 작업        → Haiku (비용 효율)       │
  └────────────────────────────────────────────────┘

  구현: heartbeat.ts에서 issue.priority + issue.type 기반으로
       runtimeConfig.model을 동적 오버라이드

전략 2: 실패 시 모델 에스컬레이션
  ┌────────────────────────────────────────────────┐
  │  Haiku 실패 → Sonnet 재시도 → Opus 재시도        │
  │                                                 │
  │  exitCode != 0 또는 작업 미완료 시               │
  │  더 강한 모델로 자동 에스컬레이션                  │
  └────────────────────────────────────────────────┘

  구현: heartbeat.ts 후처리에서 실패 감지 시
       forceFreshSession + 상위 모델로 재실행

전략 3: Agent 역할별 모델 프리셋
  ┌────────────────────────────────────────────────┐
  │  CEO (전략)       → Opus (복잡한 의사결정)        │
  │  Architect (설계)  → Opus (아키텍처 판단)         │
  │  Engineer (구현)   → Sonnet (코드 작성 균형)      │
  │  Reviewer (리뷰)   → Sonnet (코드 분석)          │
  │  Documenter (문서) → Haiku (반복 작업 효율)       │
  └────────────────────────────────────────────────┘

전략 4: A/B 테스트 프레임워크
  → 동일 작업을 다른 모델로 실행하여 품질/비용 비교
  → Config Revision + CostEvent 데이터로 분석
  → 최적 모델-역할 매핑 도출
```

---

## Part 5: 전략적 개선 로드맵

현재 시스템의 한계를 극복하기 위한 개선 영역을 우선순위별로 정리한다.

### 5.1 즉시 개선 가능 (현재 구조 내)

```
[P0] Agent Instructions 최적화
  ├── 현재: AGENTS.md 하나로 모든 지시
  ├── 개선: 역할별 모듈화 (base.md + role-engineer.md + project-context.md)
  ├── 효과: 컨텍스트 효율 향상, 불필요한 지시 제거
  └── 방법: instructionsEntryFile을 역할별로 분기

[P0] Session Compaction Policy 튜닝
  ├── 현재: 기본값 (200 runs, 2M tokens, 72시간)
  ├── 개선: 역할/모델별 최적 값 탐색
  ├── 효과: 컨텍스트 신선도 향상, 비용 절감
  └── 방법: runtimeConfig.heartbeat.sessionCompaction 조정

[P1] 평가 메트릭 확장
  ├── 현재: 비용 + 토큰만 추적
  ├── 개선: 완료율, 소요시간, 재작업률, PR 승인율 추적
  ├── 효과: Agent 성능 가시성 확보
  └── 방법: heartbeat 후처리에 메트릭 계산 로직 추가
```

### 5.2 중기 개선 (구조 변경 필요)

```
[P1] Agent 간 직접 통신 채널
  ├── 현재: Issue Comment만 (비동기, 느림)
  ├── 개선: Agent-to-Agent 메시지 큐 또는 이벤트 버스
  ├── 효과: 피드백 루프 단축, 실시간 협업 가능
  └── 방법: AgentMessage 테이블 + Wakeup 트리거 연동

[P1] 모델 라우팅 엔진
  ├── 현재: Agent별 고정 모델
  ├── 개선: 작업 복잡도 기반 동적 모델 선택
  ├── 효과: 비용 30-50% 절감 + 품질 유지
  └── 방법: heartbeat.ts에 모델 라우팅 미들웨어 삽입

[P2] 자동 상태 전환 규칙
  ├── 현재: Agent가 수동으로 상태 변경
  ├── 개선: PR 머지 → auto-done, 테스트 실패 → auto-blocked
  ├── 효과: Heartbeat 사이클 단축, 자동화 수준 향상
  └── 방법: Webhook/Plugin으로 Git 이벤트 수신 → Issue 상태 자동 전환
```

### 5.3 장기 개선 (아키텍처 확장)

```
[P2] Multi-Agent 협업 패턴
  ├── 현재: 단일 Assignee, 순차 실행
  ├── 개선: 코드 리뷰, 페어 프로그래밍, 합의 기반 의사결정
  ├── 효과: 코드 품질 향상, 오류 감소
  └── 방법: Issue에 reviewer_agent_id 추가, Review Workflow 구현

[P3] 자율적 에이전트 평가 & 개선
  ├── 현재: 정적 평가 (Promptfoo 8개 테스트)
  ├── 개선: 실행 중 자동 품질 평가, 프롬프트 자동 최적화
  ├── 효과: 자기 개선하는 에이전트 시스템
  └── 방법: Eval Agent 추가 (다른 Agent의 산출물을 자동 평가)

[P3] 지식 기반 구축
  ├── 현재: 세션 종료 시 컨텍스트 소실 (핸드오프만)
  ├── 개선: 에이전트별 영구 메모리 (para-memory-files 스킬 확장)
  ├── 효과: 장기적 학습 효과, 반복 작업 효율 향상
  └── 방법: Agent Home에 구조화된 메모리 파일 관리
```

---

## Part 6: 핵심 코드 위치 참조

| 관심 영역 | 파일 | 핵심 함수/섹션 |
|-----------|------|--------------|
| 프롬프트 구성 | `server/src/services/agent-instructions.ts` | `listFilesRecursive()`, bundle 로딩 |
| 프롬프트 렌더링 | `packages/adapters/claude-local/src/server/execute.ts:312-430` | `templateData`, `joinPromptSections()` |
| 스킬 마운팅 | `packages/adapters/claude-local/src/server/execute.ts:41-60` | `buildSkillsDir()` |
| Heartbeat 절차 | `skills/paperclip/SKILL.md` | 9단계 Heartbeat Procedure |
| 컨텍스트 빌드 | `server/src/services/heartbeat.ts:2374-2441` | context 객체 구성 |
| 세션 컴팩션 | `packages/adapter-utils/src/session-compaction.ts` | `SessionCompactionPolicy` |
| 평가 테스트 | `evals/promptfoo/tests/core.yaml` | 6개 핵심 행동 테스트 |
| 모델 설정 | `packages/adapters/claude-local/src/server/execute.ts:316` | `config.model` |
| 비용 추적 | `server/src/services/heartbeat.ts:1927-1980` | `updateRuntimeState()` |
| Config 버전 관리 | `server/src/services/agents.ts` | `agentConfigRevisions` |
| Agent API 인증 | `server/src/middleware/` | JWT, API Key 미들웨어 |
