# Work Item (Issue) Flow

## 1. Issue 데이터 모델

Issue는 Paperclip의 핵심 작업 단위(Work Item)이다.

```typescript
Issue {
  id: UUID
  companyId: UUID
  identifier: string        // "PAP-123" (unique, 자동 생성)

  // 계층 구조
  projectId: UUID | null
  goalId: UUID | null
  parentId: UUID | null     // 부모 Issue (Epic → Task 구조)

  // 내용
  title: string
  description: string       // 수락 기준, 상세 컨텍스트
  priority: enum            // critical | high | medium | low

  // 할당 & 상태
  assigneeAgentId: UUID | null
  assigneeUserId: UUID | null
  status: enum              // backlog | todo | in_progress | in_review | blocked | done | cancelled

  // 실행 추적
  checkoutRunId: UUID | null        // 체크아웃한 HeartbeatRun
  executionRunId: UUID | null       // 현재 실행 중인 Run
  executionLockedAt: timestamp      // 잠금 시간
  executionWorkspaceId: UUID | null // 실행 환경

  // 타임라인
  startedAt: timestamp | null
  completedAt: timestamp | null
  cancelledAt: timestamp | null
  hiddenAt: timestamp | null

  // 비용 귀속
  billingCode: string | null  // 상위 작업으로 비용 롤업용
}
```

## 2. Issue 상태 머신

```
                    할당
  backlog ─────────→ todo
                      │
              Agent   │
              체크아웃  ▼
                   in_progress ──────→ in_review ──────→ done
                      │                   │
                      │    Board          │  Board
                      │    블로킹          │  리젝트
                      ▼                   ▼
                   blocked            in_progress
                      │                (피드백 반영)
                      ▼
                   cancelled
```

### 상태 전환 상세

| 전환 | 트리거 | 조건 |
|------|--------|------|
| `backlog` → `todo` | Agent/Board가 할당 | assigneeAgentId 설정 |
| `todo` → `in_progress` | Agent의 atomic checkout | 다른 Agent가 점유하지 않은 상태 |
| `in_progress` → `in_review` | Agent가 작업 완료 보고 | Work Product (PR 등) 제출 |
| `in_review` → `done` | Board 승인 또는 리뷰 통과 | - |
| `in_progress` → `blocked` | Board 개입 또는 의존성 문제 | - |
| 모든 상태 → `cancelled` | Board 또는 Agent 취소 | - |

## 3. Atomic Checkout 메커니즘 (이중 작업 방지)

Paperclip에서 가장 중요한 동시성 제어 장치이다. DB 레벨의 원자적 UPDATE로 구현된다.

### 체크아웃 SQL 로직

```sql
UPDATE issues
SET
  assigneeAgentId = $agentId,
  checkoutRunId = $checkoutRunId,
  executionRunId = $checkoutRunId,
  executionLockedAt = NOW(),
  status = 'in_progress'
WHERE
  id = $issueId
  AND status IN ('todo', 'backlog')   -- 예상 상태만 허용
  AND (
    assigneeAgentId IS NULL           -- 아직 할당 안됨
    OR (
      assigneeAgentId = $agentId      -- 같은 에이전트가 재시도
      AND (checkoutRunId IS NULL OR checkoutRunId = $checkoutRunId)
    )
  )
  AND (
    executionRunId IS NULL            -- 실행 잠금 없음
    OR executionRunId = $checkoutRunId -- 같은 Run의 재시도
  )
```

### 이중 방어 구조

```
┌─────────────────────────────────────────────────┐
│  1차 방어: assigneeAgentId                        │
│  → 이미 다른 Agent에게 할당된 Issue은 체크아웃 불가   │
├─────────────────────────────────────────────────┤
│  2차 방어: executionRunId                         │
│  → 이미 다른 HeartbeatRun이 실행 중이면 체크아웃 불가 │
├─────────────────────────────────────────────────┤
│  3차 방어: status 검증                             │
│  → todo/backlog 상태가 아니면 체크아웃 불가           │
└─────────────────────────────────────────────────┘
```

### 실패 시 409 Conflict 반환

UPDATE의 영향 행이 0이면, 다른 Agent/Run이 먼저 점유한 것이다. 호출자에게 409를 반환하여 재시도 또는 다른 작업을 선택하도록 한다.

### 충돌 복구 (Stale Checkout Adoption)

이전 Run이 비정상 종료된 경우:

```
adoptStaleCheckoutRun():
  1. 같은 Agent인지 확인
  2. Issue가 여전히 in_progress인지 확인
  3. 이전 Run의 checkout을 현재 Run이 인수
  4. processLossRetryCount 증가 (재시도 추적)
```

## 4. 계층적 작업 모델

```
[Company Goal] "AI 챗봇 서비스 런칭"
  │
  ├── [Project] "AI Chat 개발"
  │     │
  │     ├── [Epic/Parent Issue] "인증 시스템"
  │     │     ├── [Task] "OAuth 구현"        → 할당: engineer-1
  │     │     └── [Task] "비밀번호 리셋"      → 할당: engineer-2
  │     │
  │     └── [Epic/Parent Issue] "채팅 UI"
  │           ├── [Task] "채팅 레이아웃"       → 할당: designer-1
  │           └── [Task] "LLM API 연동"       → 할당: engineer-1
  │
  └── [Project] "마케팅 사이트"
        └── [Task] "랜딩페이지 디자인"          → 할당: designer-1
```

### 크로스팀 작업

- 같은 프로젝트가 아닌 다른 팀의 Agent에게도 작업 할당 가능
- `billingCode` 를 통해 비용을 원래 요청자에게 귀속 가능

## 5. Issue Comment (에이전트 간 통신)

```typescript
IssueComment {
  id: UUID
  issueId: UUID
  authorAgentId: UUID | null   // Agent가 작성
  authorUserId: UUID | null    // Board가 작성
  body: string                 // 마크다운 본문
  createdAt: timestamp
}
```

**현재 통신 방식의 특징:**

- 모든 Agent-Agent, Agent-Board 소통은 Issue Comment를 통해 이루어짐
- 비동기적 (다음 Heartbeat에서 확인)
- 불변(Immutable) 감사 추적
- 실시간 채팅은 불가

## 6. Work Product (산출물 추적)

Agent가 작업 중 생산한 결과물을 추적한다.

```typescript
IssueWorkProduct {
  id: UUID
  issueId: UUID
  type: string           // "pull_request", "deployment", "document", "design"
  provider: string       // "github", "vercel", "figma"
  externalId: string     // 외부 시스템 ID
  title: string
  url: string            // PR 링크, 배포 URL 등
  status: string         // "open", "merged", "deployed"
  reviewState: string    // "pending", "approved", "changes_requested"
  isPrimary: boolean     // 주요 산출물 여부
}
```

### 일반적인 산출물 흐름

```
1. Agent가 코드 작성
2. Agent가 PR 생성 → IssueWorkProduct (type: "pull_request", status: "open")
3. Board/다른 Agent가 리뷰
4. PR 머지 → status: "merged", reviewState: "approved"
5. 배포 → 새 WorkProduct (type: "deployment", url: "https://...")
6. Issue → done
```

## 7. Execution Workspace (실행 환경)

Agent가 코드를 실행할 물리적 작업 공간.

### 워크스페이스 모드

| 모드 | 설명 | 사용 시나리오 |
|------|------|-------------|
| `shared_workspace` | 프로젝트 전체에서 하나의 Git 저장소 공유 | 동일 프로젝트 작업 시 |
| `task_session` | 각 Issue별 독립 워크스페이스 | 격리가 필요한 작업 |

### 워크스페이스 전략

| 전략 | 설명 |
|------|------|
| `project_primary` | 프로젝트 메인 Git 저장소 사용 |
| `git_worktree` | Git worktree로 Issue별 브랜치 격리 |

### 워크스페이스 결정 로직

```
resolveWorkspaceForRun():
  1. Issue에 preferredWorkspaceId가 있으면 → 그것 사용
  2. Project에 default workspace가 있으면 → 그것 사용
  3. Task 격리가 필요하면 → 임시 워크스페이스 생성
  4. 위 모두 해당 없으면 → Agent Home (~/.paperclip/agents/{agentId}/)
```

### 워크스페이스 프로비저닝

```
ensureManagedProjectWorkspace():
  1. cwd가 존재하고 Git 저장소인지 확인
  2. 아니면:
     a. git clone repoUrl cwd (타임아웃: 10분)
     b. workspaceHints에 저장 (Agent 컨텍스트용)
  3. Git worktree 전략이면:
     a. git worktree add -b <브랜치> <경로>
     b. executionWorkspaces.providerRef에 기록
```

## 8. Issue 전체 흐름 예시

```
[Board] Issue #123 생성: "로그인 폼 구현"
  ├── title: "Build login form"
  ├── assigneeAgentId: engineer-1
  ├── status: todo
  ├── projectId: "ai-chat"
  │
  ▼
[Heartbeat] engineer-1 깨어남 (스케줄 또는 즉시)
  ├── todo 상태 Issue 조회 → #123 발견
  ├── Atomic Checkout 시도 → 성공 (status → in_progress)
  │
  ▼
[Adapter] Claude Code에 컨텍스트 전달:
  {
    issue: { id: "123", title: "Build login form", ... },
    workspace: { cwd: "/project/ai-chat", repoUrl: "..." },
    context: { companyGoal: "...", project: "..." }
  }
  │
  ▼
[Agent 실행]
  ├── 저장소 클론 (필요시)
  ├── 코드 구현
  ├── 테스트 실행
  ├── PR #42 생성 → IssueWorkProduct 생성
  ├── Comment: "Form implemented, PR #42"
  │
  ▼
[Heartbeat 완료]
  ├── CostEvent 기록 (토큰 사용량, 비용)
  ├── HeartbeatRun 결과 저장
  ├── Session 상태 persist
  │
  ▼
[Board 리뷰]
  ├── PR #42 확인 → 승인 & 머지
  ├── Issue #123 → done
  └── 비용 확인: $0.15 (10K input + 2K output tokens)
```

## 9. 설계 시사점 & 한계

### 강점
- **Atomic Checkout**: DB 레벨 동시성 제어로 이중 작업 완벽 방지
- **계층적 작업 모델**: 회사 목표부터 개별 태스크까지 추적 가능
- **Work Product 추적**: PR, 배포, 문서 등 산출물을 Issue에 연결

### 개선 가능 포인트
- **Issue Comment 기반 통신만 가능**: 에이전트 간 실시간 대화나 직접 메시지 불가. 모든 소통이 다음 Heartbeat를 기다려야 함.
- **피드백 루프가 느림**: Board 리뷰 → Comment → 다음 Heartbeat에서 확인 → 수정. 여러 턴의 피드백에 시간이 많이 걸림.
- **단일 Assignee 제한**: 하나의 Issue에 여러 Agent가 동시 작업할 수 없음. 코드 리뷰, 페어 작업 패턴 구현 어려움.
- **우선순위 기반 작업 선택 로직 부재**: Agent가 여러 todo Issue를 가진 경우, 어떤 것을 먼저 처리할지의 지능적 선택이 명시적이지 않음.
- **Issue 상태 전환이 수동적**: Agent가 스스로 상태를 변경해야 함. 자동 전환 규칙(예: PR 머지 시 자동 done)이 없음.
