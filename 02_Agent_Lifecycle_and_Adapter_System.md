# Agent Lifecycle & Adapter System

## 1. Agent 데이터 모델

```typescript
Agent {
  id: UUID
  companyId: UUID
  name: string              // 표시 이름 (중복 시 자동 suffix)
  role: string              // "ceo", "engineer", "designer" 등
  title: string             // 직함
  icon: string              // 아이콘
  reports_to: UUID | null   // 상위 매니저 (null = CEO/최상위)

  // 실행 설정
  adapterType: string       // "claude_local", "codex_local", "http" 등
  adapterConfig: JSONB      // 어댑터별 설정 (모델, 명령어 등)
  runtimeConfig: JSONB      // 런타임 설정 (timeout, cwd 등)

  // 상태
  status: enum              // pending_approval → idle → busy → paused
  pauseReason: string | null
  lastHeartbeatAt: timestamp

  // 예산
  budgetMonthlyCents: number
  spentMonthlyCents: number

  // 메타
  metadata: JSONB
  permissions: JSONB
}
```

## 2. Agent 생명주기 상태 머신

```
                  Board 승인
  pending_approval ──────────→ idle
                                │
                     Heartbeat  │  작업 완료
                     트리거     ↓     │
                              busy ───┘
                                │
                     예산 초과   │  Board 정지
                     or 오류    ↓
                              paused
                                │
                     Board 재개  │  예산 증액
                                ↓
                              idle
```

### 상태 전환 규칙

| 현재 상태 | → 다음 상태 | 트리거 |
|-----------|------------|--------|
| `pending_approval` | `idle` | Board가 `activatePendingApproval()` 호출 |
| `idle` | `busy` | Heartbeat 실행 시작 |
| `busy` | `idle` | Heartbeat 실행 완료 |
| `idle/busy` | `paused` | 예산 초과(hard-stop), Board 수동 정지, 오류 |
| `paused` | `idle` | Board 재개, 예산 증액 후 `resumeScopeFromBudget()` |

## 3. Agent 생성 플로우

```
1. Board가 에이전트 생성 요청 (UI 또는 CEO 에이전트가 hire 요청)
   │
2. create() 함수 실행
   ├── 이름 중복 검사 → deduplicateAgentName() (suffix 추가)
   ├── URL-safe shortname 생성 → normalizeAgentUrlKey()
   ├── 조직 계층 검증 → assertNoCycle() (순환 참조 방지)
   ├── 역할 기반 권한 설정
   └── status = 'pending_approval' 로 저장
   │
3. Board 승인 대기
   │
4. Board 승인 → activatePendingApproval()
   ├── status → 'idle'
   ├── API Key 발급 가능
   └── 어댑터 onHireApproved() 훅 호출 (있는 경우)
   │
5. Agent 활성화 완료 → Heartbeat 대상에 포함
```

### 조직 구조 규칙

- **Strict Tree**: 모든 Agent는 정확히 하나의 매니저에게 보고 (reports_to)
- **No Cycles**: `assertNoCycle()` 로 순환 참조 검증
- **CEO = Root**: `reports_to = null`인 Agent가 최상위
- **Config Revision Tracking**: 설정 변경 시 before/after 스냅샷 저장 (`agentConfigRevisions`)

## 4. Adapter 인터페이스

모든 어댑터는 `ServerAdapterModule` 인터페이스를 구현해야 한다.

### 필수 메서드

```typescript
interface ServerAdapterModule {
  type: string;

  // [필수] 에이전트 실행
  execute(ctx: AdapterExecutionContext): Promise<AdapterExecutionResult>;

  // [필수] 환경 검증 (설정이 유효한지 테스트)
  testEnvironment(ctx: AdapterEnvironmentTestContext): Promise<AdapterEnvironmentTestResult>;
}
```

### 선택 메서드

```typescript
interface ServerAdapterModule {
  // 세션 직렬화/역직렬화 (상태 유지에 필요)
  sessionCodec?: AdapterSessionCodec;

  // 스킬 관리
  listSkills?(ctx): Promise<AdapterSkillSnapshot>;
  syncSkills?(ctx, desiredSkills): Promise<AdapterSkillSnapshot>;

  // 생명주기 훅
  onHireApproved?(payload, adapterConfig): Promise<HireApprovedHookResult>;

  // 할당량 관리 (API 레이트 리밋)
  getQuotaWindows?(): Promise<ProviderQuotaResult>;

  // 모델 정보
  detectModel?(): Promise<{ model, provider, source } | null>;
  listModels?(): Promise<AdapterModel[]>;
}
```

### execute() 입력/출력

**입력 (AdapterExecutionContext):**

```typescript
{
  runId: string              // HeartbeatRun UUID
  agent: {
    id, companyId, name,
    adapterType, adapterConfig
  }
  runtime: {
    sessionId,               // 이전 세션 ID (resume용)
    sessionParams,           // 어댑터별 세션 상태
    sessionDisplayId,
    taskKey                  // 작업 식별자
  }
  config: Record<string, unknown>   // runtimeConfig
  context: Record<string, unknown>  // 회사/작업 컨텍스트

  // 콜백
  onLog: (stream, chunk) => Promise<void>     // 실시간 로그
  onMeta?: (meta) => Promise<void>            // 메타데이터
  onSpawn?: (meta: { pid, startedAt }) => Promise<void>  // 프로세스 시작
  authToken?: string                          // 인증 토큰
}
```

**출력 (AdapterExecutionResult):**

```typescript
{
  exitCode: number | null
  signal: string | null
  timedOut: boolean
  errorMessage?: string

  // 사용량 & 비용
  usage?: { inputTokens, outputTokens, cachedInputTokens }
  provider?: string          // "anthropic", "openai"
  biller?: string
  model?: string             // "claude-opus", "gpt-4"
  billingType?: string       // "metered_api", "subscription_included"
  costUsd?: number

  // 세션 관리
  sessionId?: string
  sessionParams?: Record<string, unknown>
  sessionDisplayId?: string
  clearSession?: boolean     // true면 세션 초기화

  // 결과
  resultJson?: Record<string, unknown>
  summary?: string
  runtimeServices?: AdapterRuntimeServiceReport[]
}
```

## 5. 어댑터 종류별 비교

| 어댑터 | 실행 방식 | 세션 유지 | 주요 특징 |
|--------|----------|----------|----------|
| `claude_local` | 로컬 CLI 프로세스 | O (sessionId) | `--resume` 으로 세션 복구, `--append-system-prompt-file`로 지시 주입 |
| `codex_local` | 로컬 CLI 프로세스 | O | Codex CLI 호출 |
| `cursor_local` | Cursor IDE 브릿지 | O | Cursor API 연동 |
| `gemini_local` | 로컬 프로세스 | O | Google Gemini |
| `openclaw_gateway` | Gateway 매개 | O | 관리형 OpenClaw |
| `process` (내장) | 자식 프로세스 spawn | X | `execFile('node agent.js')` |
| `http` (내장) | HTTP POST webhook | X | 외부 서비스 호출 |

## 6. Claude Local 어댑터 상세

가장 핵심적인 어댑터. Claude Code CLI를 호출하여 에이전트를 실행한다.

### 호출 파라미터

```bash
claude \
  --print - \
  --output-format stream-json \
  --verbose \
  --resume <sessionId> \           # 세션 복구 (있는 경우)
  --model <model> \                # claude-opus 등
  --effort <effort> \              # thinking effort level
  --max-turns <N> \                # 최대 대화 턴 수
  --chrome \                       # 브라우저 자동화 활성화
  --dangerously-skip-permissions \ # 권한 확인 스킵
  --append-system-prompt-file <path> \  # 에이전트 지시 파일
  --add-dir <skillsDir>            # 스킬 디렉토리
```

### 주입되는 환경 변수

```
PAPERCLIP_RUN_ID          - 현재 실행 UUID
PAPERCLIP_AGENT_ID        - 에이전트 ID
PAPERCLIP_AGENT_NAME      - 에이전트 이름
PAPERCLIP_COMPANY_ID      - 회사 ID
PAPERCLIP_API_KEY         - 인증 토큰
PAPERCLIP_WORKSPACE_CWD   - 작업 디렉토리
PAPERCLIP_WORKSPACE_ID    - 워크스페이스 ID
PAPERCLIP_WORKSPACE_REPO_URL - Git 저장소 URL
PAPERCLIP_TASK_ID         - 작업 ID
PAPERCLIP_APPROVAL_ID     - 승인 요청 ID
PAPERCLIP_WAKE_REASON     - 깨어난 이유
PAPERCLIP_LINKED_ISSUE_IDS - 연결된 이슈 ID 목록
```

### 세션 관리

```
1. 실행 시작
   ├── 이전 sessionId 존재? → --resume sessionId 로 복구 시도
   │   ├── 복구 성공 → 이전 대화 이어서 진행
   │   └── "unknown session" 오류 → 자동으로 새 세션 시작
   └── 없음 → 새 세션 생성

2. 실행 완료
   ├── sessionParams에 sessionId, cwd 등 저장
   └── AgentTaskSession 테이블에 persist

3. 세션 컴팩션 (Session Compaction)
   ├── 토큰 임계값 초과, 세션 수명 초과, 실행 횟수 초과 시
   └── 자동으로 새 세션으로 전환 (컨텍스트 신선도 유지)
```

## 7. Agent Instructions (프롬프트 구성)

에이전트가 실행될 때 시스템 프롬프트가 어떻게 구성되는지:

### Instruction Bundle 모드 (우선순위 순)

1. **Managed Bundle** (권장)
   ```
   companies/{companyId}/agents/{agentId}/instructions/
   ├── AGENTS.md          # 엔트리 파일
   ├── project-context.md
   └── guidelines.json
   ```

2. **External Bundle**
   - `instructionsRootPath`로 지정한 커스텀 디렉토리

3. **Legacy** (deprecated)
   - 단일 `instructionsFilePath` 또는 인라인 `promptTemplate`

### 프롬프트 변수 치환

```
{agent.id}        → 에이전트 UUID
{agent.name}      → 에이전트 이름
{agent.companyId} → 회사 ID
{company.id}      → 회사 ID
{runId}           → 현재 실행 ID
{context.*}       → 동적 컨텍스트
```

## 8. Agent 권한 모델

| 역할 | 권한 |
|------|------|
| CEO | 목표 수정, 작업 위임, 하위 에이전트 예산 설정, 전략 제안 |
| Manager | 하위 에이전트에 작업 할당, 문제 에스컬레이션 |
| IC (Individual Contributor) | 할당된 작업 수행, 이슈 코멘트 작성 |
| Collaborator | 크로스팀 이슈 참여 |

## 9. 설계 시사점 & 한계

### 강점
- **어댑터 패턴**: 새로운 AI 런타임 추가가 쉬움 (인터페이스만 구현)
- **세션 지속성**: 에이전트가 작업을 이어서 할 수 있음
- **Config Revision**: 설정 변경 이력 추적 가능

### 개선 가능 포인트
- **에이전트 간 직접 통신 없음**: 모든 소통이 Issue Comment를 통해 비동기적으로만 가능. 실시간 협업이 불가.
- **단일 Assignee 모델**: 하나의 Issue에 하나의 에이전트만 할당 가능. 페어 프로그래밍이나 동시 작업 패턴 불가.
- **역할 기반 권한이 느슨함**: JSONB로 저장되어 강제(enforcement)가 서비스 레벨에서만 이루어짐.
- **어댑터 간 호환성 차이**: Claude Local은 세션/스킬/프롬프트 주입이 풍부하지만, HTTP 어댑터는 매우 단순. 어댑터별 기능 격차가 큼.
