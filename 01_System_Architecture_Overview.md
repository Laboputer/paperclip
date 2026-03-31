# Paperclip - System Architecture Overview

## 1. 프로젝트 개요

Paperclip은 AI Agent들로 구성된 자율 회사를 운영하는 **제어 플레인(Control Plane)** 시스템이다. 핵심 철학은 다음과 같다:

- **Control Plane, Not Execution Plane**: Paperclip은 작업을 관리하고 조율할 뿐, 실제 실행은 외부 에이전트(Claude Code, Codex, Cursor 등)가 수행한다.
- **Company-First Model**: 모든 데이터는 Company 단위로 격리된다. 하나의 Paperclip 인스턴스에서 복수의 독립 회사를 운영할 수 있다.
- **Board Governance**: 인간 운영자("Board")가 최종 통제권을 가진다. 에이전트 채용, 예산, 전략 승인 등 핵심 의사결정에 관여한다.
- **Adapter Agnostic**: 에이전트 런타임에 비종속적이다. Claude Code, Codex, Cursor, HTTP webhook 등 어떤 실행 환경이든 어댑터만 구현하면 된다.

## 2. 기술 스택

| 구분 | 기술 |
|------|------|
| Language | TypeScript (전체) |
| Server | Express.js REST API |
| Database | PostgreSQL 17 (Drizzle ORM) |
| Frontend | React + Vite |
| CLI | Node.js CLI (`pnpm paperclipai`) |
| Package Manager | pnpm (workspace monorepo) |
| Container | Docker + docker-compose |
| Test | Vitest |

## 3. 모노레포 패키지 구조

```
paperclip/
├── server/              # Express REST API 서버 (메인 제어 플레인)
│   └── src/
│       ├── routes/      # 60+ HTTP 라우트 모듈
│       ├── services/    # 70+ 서비스 모듈 (비즈니스 로직 핵심)
│       ├── middleware/   # 인증, 로깅, 에러 핸들링
│       └── lib/         # 유틸리티, 헬퍼
│
├── packages/
│   ├── db/              # Drizzle ORM 스키마 (59개 테이블)
│   │   └── src/schema/  # 테이블 정의 파일들
│   ├── shared/          # 공유 타입, API 계약, 상수, 밸리데이터
│   ├── adapters/        # Agent 런타임 어댑터 (8종)
│   │   ├── claude-local/     # Claude Code 터미널
│   │   ├── codex-local/      # Codex CLI
│   │   ├── cursor-local/     # Cursor IDE
│   │   ├── gemini-local/     # Google Gemini
│   │   ├── opencode-local/   # OpenCode
│   │   ├── pi-local/         # Pi
│   │   ├── openclaw-gateway/ # OpenClaw 관리형
│   │   └── hermes/           # Hermes
│   ├── adapter-utils/   # 어댑터 공통 인터페이스, 유틸리티
│   └── plugins/         # 플러그인 SDK (확장 시스템)
│
├── ui/                  # React 프론트엔드 (Board 대시보드)
├── cli/                 # CLI 도구 (onboard, 회사 관리)
├── skills/              # Agent에게 제공하는 스킬 파일들
├── scripts/             # 빌드, 배포, 유틸리티 스크립트
├── evals/               # 평가 테스트
├── tests/               # 통합 테스트
└── docker/              # Docker 설정 파일
```

## 4. 데이터 흐름 개요

```
┌────────────────────────────────────────────────────────────────┐
│  Board (인간 운영자) - UI Dashboard                              │
│  - 회사 설정, 에이전트 채용/해고, 예산 관리, 승인/거절              │
└───────────────────────────┬────────────────────────────────────┘
                            │ HTTP REST API
                            ▼
┌────────────────────────────────────────────────────────────────┐
│  Server (Express)                                               │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐    │
│  │  Routes (60+)│→│ Services(70+)│→│ DB (PostgreSQL, 59T) │    │
│  └──────────────┘ └──────┬───────┘ └──────────────────────┘    │
│                          │                                      │
│  ┌───────────────────────▼──────────────────────────┐          │
│  │  Heartbeat Engine (핵심 오케스트레이션)              │          │
│  │  - Wakeup Queue 관리                              │          │
│  │  - Adapter 호출                                   │          │
│  │  - 세션 관리 & 비용 추적                            │          │
│  └───────────────────────┬──────────────────────────┘          │
└──────────────────────────┼─────────────────────────────────────┘
                           │ Adapter 호출
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   ┌────────────┐  ┌────────────┐  ┌────────────┐
   │ Claude Code │  │   Codex    │  │  HTTP/기타  │
   │  (로컬CLI)  │  │ (로컬CLI)  │  │ (Webhook)  │
   └────────────┘  └────────────┘  └────────────┘
```

## 5. 핵심 서비스 모듈 (server/src/services/)

### 오케스트레이션 관련
| 파일 | 크기 | 역할 |
|------|------|------|
| `heartbeat.ts` | ~144KB | **핵심 엔진**. Agent 깨우기, 어댑터 호출, 실행 결과 기록 |
| `agent-instructions.ts` | ~26KB | Agent 시스템 프롬프트 구성 |
| `agents.ts` | ~24KB | Agent CRUD, 조직 구조, 상태 관리 |
| `issues.ts` | ~70KB | Work item 생명주기, atomic checkout |
| `execution-workspaces.ts` | ~25KB | 실행 워크스페이스 프로비저닝 |

### 스케줄링 & 루틴
| 파일 | 크기 | 역할 |
|------|------|------|
| `routines.ts` | ~48KB | 반복 작업 스케줄링, webhook 트리거 |
| `cron.ts` | ~11KB | Cron 표현식 파싱, 타임존 처리 |

### 재무 & 거버넌스
| 파일 | 크기 | 역할 |
|------|------|------|
| `costs.ts` | ~16KB | 비용 이벤트 기록, 집계 |
| `budgets.ts` | ~32KB | 예산 정책, hard-stop 강제 |
| `approvals.ts` | ~9KB | Board 승인 워크플로우 |
| `activity-log.ts` | ~3KB | 감사 로그 |

### 플러그인 시스템 (12개 모듈)
| 파일 | 역할 |
|------|------|
| `plugin-loader.ts` | 플러그인 로드 & 검증 |
| `plugin-worker-manager.ts` | 워커 프로세스 생성 |
| `plugin-job-scheduler.ts` | 플러그인 작업 스케줄링 |
| `plugin-event-bus.ts` | 플러그인 간 이벤트 메시징 |
| `plugin-tool-dispatcher.ts` | 도구 호출 디스패치 |

### 기타 주요 서비스
| 파일 | 역할 |
|------|------|
| `companies.ts` | 회사 생명주기 |
| `company-portability.ts` (~168KB) | 회사 전체 export/import |
| `company-skills.ts` (~84KB) | 스킬 매니페스트, 배포 |

## 6. DB 스키마 핵심 엔터티 관계

```
Company (회사)
 ├── Agent (에이전트)
 │    ├── reports_to → Agent (조직 트리)
 │    ├── HeartbeatRun (실행 기록)
 │    ├── AgentTaskSession (세션 상태)
 │    ├── AgentRuntimeState (런타임 상태)
 │    ├── AgentApiKey (인증 키)
 │    └── CostEvent (비용 기록)
 │
 ├── Goal (목표, 계층적)
 │    └── Project (프로젝트)
 │         ├── Issue (작업 = Work Item)
 │         │    ├── IssueComment (댓글)
 │         │    ├── IssueWorkProduct (산출물: PR, 배포 등)
 │         │    ├── IssueApproval (승인 요청)
 │         │    └── IssueAttachment (첨부)
 │         ├── ProjectWorkspace (실행 환경)
 │         └── ExecutionWorkspace (작업별 워크스페이스)
 │
 ├── Routine (반복 작업)
 │    ├── RoutineTrigger (cron/webhook/event)
 │    └── RoutineRun (실행 이력)
 │
 ├── Approval (거버넌스 승인)
 ├── BudgetPolicy (예산 정책)
 ├── BudgetIncident (예산 초과 이벤트)
 ├── CompanySecret (암호화된 비밀 키)
 ├── ActivityLog (감사 로그)
 └── Plugin (확장 플러그인)
```

## 7. 배포 모드

| 모드 | 인증 | 노출 | 용도 |
|------|------|------|------|
| `local_trusted` | 없음 | 로컬 | 개인 개발 |
| `authenticated` + `private` | JWT/OAuth | 사내망 | 팀 운영 |
| `authenticated` + `public` | JWT/OAuth | 공개 | 클라우드 배포 |

## 8. 시스템 불변 조건 (Invariants)

1. **Company Isolation**: 회사 간 데이터 누출 없음
2. **Atomic Task Checkout**: DB 레벨 잠금으로 이중 작업 방지
3. **Immutable Audit Log**: 모든 변경 사항 감사 로그 기록
4. **Budget Hard-Stop**: 예산 초과 시 즉시 에이전트 일시정지
5. **Persistent Sessions**: 에이전트는 중단 지점에서 재개 가능
6. **Single Assignee**: 하나의 Issue에는 하나의 에이전트만 할당
7. **Goal Ancestry**: 모든 작업은 회사 미션까지 추적 가능
8. **Board Governance**: 인간이 최종 통제권 보유, 에이전트 스스로 해고 불가
