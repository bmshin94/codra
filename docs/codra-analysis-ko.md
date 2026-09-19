# Codra 분석 및 활용 정리 (한국어)

> 이 문서는 Codra 저장소를 직접 분석하고 나눈 대화를 정리한 자료입니다.
> 작성일: 2026-09-19

---

## 🔗 관련 링크

| 구분 | 주소 |
|---|---|
| 내 저장소 (포크) | https://github.com/bmshin94/codra |
| 원본 저장소 (업스트림) | https://github.com/devarshishimpi/codra |
| 공식 웹사이트 | https://codra.run |
| 공식 문서 | https://codra.run/docs |
| 설치 가이드 | https://codra.run/docs/installation |
| 설정 가이드 | https://codra.run/docs/configuration |
| Neon 배포 가이드 | https://codra.run/docs/neon |
| Railway 배포 가이드 | https://codra.run/docs/railway |
| 이슈 트래커 | https://github.com/devarshishimpi/codra/issues |
| 기여 가이드 | [CONTRIBUTING.md](../CONTRIBUTING.md) |
| 라이선스 (AGPL-3.0) | [LICENSE](../LICENSE) |
| 보안 정책 | [SECURITY.md](../SECURITY.md) |

---

## 1. Codra란 무엇인가

### 한 줄 요약

> **Codra = 내가 직접 소유하고 운영하는 AI 코드 리뷰 봇**

GitHub에 Pull Request를 올리면 AI가 코드를 읽고 버그/보안/성능/유지보수성 문제를 찾아
**PR에 인라인 댓글로 직접 달아주는 시스템**입니다.

CodeRabbit, Greptile, Cursor BugBot 같은 유료 SaaS의 **오픈소스 셀프호스팅 대안**입니다.

### 기본 정보

| 항목 | 내용 |
|---|---|
| 이름 / 버전 | Codra `v0.9.5` (**베타**) |
| 원작자 | Devarshi Shimpi |
| 라이선스 | **AGPL-3.0-only** (+ CLA 기반 듀얼 라이선스 모델) |
| 규모 | TS/TSX 파일 **322개**, 약 **50,800줄** |
| 배포 대상 | Cloudflare Workers |

### 핵심 가치 제안

- **리뷰 루프 전체를 소유**: GitHub App, Worker, 큐, DB, 모델 크리덴셜, 대시보드 전부 내 통제 아래
- **저장소 맥락 기반 리뷰**: 정확성, 보안, 성능, 유지보수성, 저장소 고유 패턴까지 점검
- **저장소별 세밀 설정**: 트리거, 스킵 경로, draft 처리, 멘션 리뷰, 라벨, 커스텀 룰, 리뷰 예산
- **의도적인 모델 라우팅**: 전역 기본값, 저장소별 모델 체인, 폴백, 크기 기반 오버라이드
- **운영 가시성**: job 이력, PR 발견사항, 웹훅 전달, 모델 사용량, 통계

---

## 2. 저장소 구조

npm workspaces 기반 **모노레포**입니다. `apps/`(배포 진입점)와 `packages/`(재사용 모듈)로 분리되어 있습니다.

```text
codra/
├── packages/                 # npm에 @codraoss/* 로 배포되는 모듈
│   ├── schema/               # 공통 타입 + Zod 계약 (의존성 0)
│   ├── core/                 # ★ 리뷰 엔진 본체 (플랫폼 무관, ports 뒤에 숨김)
│   ├── db/                   # PostgreSQL 저장소 계층 + 마이그레이션
│   ├── models/               # LLM 제공자 어댑터 + 모델 카탈로그
│   ├── provider-github/      # GitHub App 인증, 웹훅, OAuth, 리뷰 게시
│   ├── api/                  # Hono 라우터 (마운트 가능한 HTTP 표면)
│   └── ui/                   # React 디자인 시스템 프리미티브
│
└── apps/
    ├── worker/               # Cloudflare Worker 진입점 (바인딩 → ports 배선)
    └── dashboard/            # React SPA 대시보드
```

### 의존성 방향

```text
schema → core → { db, models, provider-github } → api → apps/worker
schema → ui → apps/dashboard
```

한 방향으로만 흐르며, `core`는 Cloudflare를 전혀 알지 못합니다.
전부 **ports(인터페이스)** 뒤에 감춰져 있고 `apps/worker`가 실제 바인딩을 주입합니다.
→ **헥사고날 아키텍처(포트 & 어댑터)**

---

## 3. 동작 흐름

```text
① 개발자가 PR 오픈 / 푸시 / ready_for_review / reopen
        ↓
② GitHub → Codra 웹훅 전송
        ↓
③ HMAC 서명 검증 + 저장소별 리뷰 설정 로드
        ↓
④ PostgreSQL에 job 저장 → Cloudflare Queues에 적재
        ↓
⑤ Worker가 job 소비 → PR diff 가져오기
        ↓
⑥ 파일 단위 LLM 리뷰 패스 실행 (모델 체인 + 폴백)
        ↓
⑦ Verify 패스 — 발견사항 재검증 + 증거 매칭 + 중복 제거
        ↓
⑧ GitHub에 인라인 댓글 + 요약 리뷰 + Check Run 게시
        ↓
⑨ 대시보드에 job 이력 / 발견사항 / 로그 / 통계 기록
```

### 부품별 역할 (비유)

| 부품 | 비유 | 실제 역할 |
|---|---|---|
| GitHub 웹훅 | 주문 전화벨 | "PR 올라왔어요" 알림 |
| PostgreSQL | 주문 장부 | job / 발견사항 / 설정 영속화 |
| Cloudflare Queues | 대기표 | 순서 보장, 유실 방지 |
| Worker | 주방 | 실제 리뷰 실행 |
| LLM | 요리사의 두뇌 | 코드 판단 |
| Verify 패스 | 검수 담당 | 오탐 제거 |
| Cloudflare KV | 포스트잇 | 빠른 플래그/캐시 |
| Dashboard | 매장 관리 화면 | 이력·통계·실패 확인 |

---

## 4. 기술 스택

| 레이어 | 기술 |
|---|---|
| Worker | Cloudflare Workers, Hono, Wrangler |
| Dashboard | React 19, Vite 8, Tailwind CSS 4, Base UI / Radix, Recharts |
| Data | PostgreSQL, Cloudflare Hyperdrive, Cloudflare KV |
| Queue | Cloudflare Queues, Cloudflare Workflows |
| Models | OpenAI, OpenRouter, Anthropic, Google, NVIDIA, Cloudflare Workers AI, Vertex |
| GitHub | GitHub App 웹훅, Checks, Reviews, OAuth |
| Quality | TypeScript, Zod, Vitest, Playwright |

---

## 5. 코드에서 발견한 인상적인 설계

실제 소스를 읽으며 확인한, 실무적으로 배울 가치가 큰 부분들입니다.

### 5-1. 서브리퀘스트 예산 관리
📁 `packages/core/src/review/budget.ts`

Cloudflare 무료 플랜의 요청당 외부 호출(subrequest) 제한을 고려해,
"파일 1개당 예상 호출 수"를 계산하여 **처리 파일 수를 동적으로 제한**합니다.

```ts
// 예산이 남아있는 한 절대 0을 반환하지 않음
// → 한 번에 1개씩이라도 진행하게 만듦
return Math.max(remainingSafeBudget > 0 ? 1 : 0, Math.min(configuredChunkFileLimit, budgetLimit));
```

2차 리뷰어를 쓰면 모델 호출이 두 배가 되므로 파일 수를 절반으로 줄이고,
나머지는 continuation 루프가 다음 호출로 이월합니다.

### 5-2. 오탐(False Positive) 제거 게이트
📁 `packages/core/src/finding-gates.ts`, `packages/core/src/verify.ts`

AI 코드 리뷰의 최대 약점인 "쓸데없는 지적 폭탄"을 막는 다단계 필터:

1. **증거 매칭** — 지적한 내용이 실제 diff에 근거가 있는지 확인
2. **저수확 제목 필터** — `missing|redundant|repetitive|inconsisten|documentation|type|any|potential` 정규식으로 저품질 지적 차단
3. **2차 모델 검증** — 별도 verify 패스로 "이게 진짜 버그인가" 재확인
4. **중복 제거** — `model-output/dedupe.ts`

> **이 부분이 Codra의 진짜 가치입니다.**
> AI를 붙이는 건 쉽지만, **봇을 조용하게 만드는 것**이 어렵습니다.

### 5-3. 큐 순차 처리 (의도적 설계)
📁 `apps/worker/src/index.ts`

```ts
// Sequential: parallel fan-out could breach the Free plan subrequest cap.
```

병렬 fan-out 대신 의도적으로 순차 처리합니다.

### 5-4. KV 플래그로 서버리스 DB 비용 절감
📁 `apps/worker/src/index.ts` (`scheduled` 핸들러)

2분마다 도는 크론에서 KV의 `system:active_jobs` 플래그가 없으면 **DB를 아예 건드리지 않습니다.**
작업이 끝나면 TTL을 기다리지 않고 플래그를 즉시 해제합니다.

### 5-5. Job 복구 (lease recovery)
📁 `apps/worker/src/core/job-recovery.ts`

Worker가 중단되어도 멈춘 job을 되살립니다.
스키마 검증에 실패한 큐 메시지는 ack 처리하되, 해당 job을 명시적으로 실패 처리해
"running 상태로만 복구되는" lease recovery의 사각지대를 메웁니다.

### 5-6. 크리덴셜 보안
- 📁 `packages/models/src/llm-crypto.ts` — LLM API 키를 `LLM_CONFIG_ENCRYPTION_KEY`로 암호화해 DB 저장
- 📁 `packages/models/src/url-guard.ts` — 커스텀 제공자 URL의 내부망(private IP) 접근 차단 (SSRF 방어)

---

## 6. 설치 및 사용법

### 6-1. 준비물

| 필요한 것 | 용도 | 비용 |
|---|---|---|
| Cloudflare 계정 | Worker / Queues / KV / Hyperdrive | Queues는 유료 플랜 필요 (약 $5/월) |
| PostgreSQL | 데이터 저장 (Neon / Railway / Supabase) | 무료 티어 가능 |
| GitHub App | 웹훅 + 리뷰 게시 권한 | 무료 |
| LLM API 키 | 실제 리뷰 수행 | 사용량 과금 |
| Node.js + npm | 빌드 / 배포 | 무료 |

### 6-2. 설치 순서

```bash
# 1. 클론 + 의존성 설치
git clone https://github.com/bmshin94/codra
cd codra
npm install

# 2. 환경변수 준비
cp .dev.vars.example .dev.vars

# 3. Cloudflare 인증
npx wrangler login

# 4. Cloudflare 리소스 자동 프로비저닝 (KV / Hyperdrive / 시크릿)
npm run setup:cloudflare

# 5. DB 마이그레이션
npm run migrate

# 6. 로컬 개발 (클라이언트 + 워커 동시 실행)
npm run dev

# 7. 배포
npm run deploy
```

### 6-3. 주요 npm 스크립트

| 명령 | 설명 |
|---|---|
| `npm run dev` | 대시보드 watch 빌드 + `wrangler dev` 동시 실행 |
| `npm run build` | Vite 빌드 + Cloudflare 타입 생성 |
| `npm run deploy` | 빌드 → 마이그레이션 → `wrangler deploy` |
| `npm run migrate` | PostgreSQL 마이그레이션 |
| `npm run test` | 테스트 실행 |
| `npm run lint` | ESLint |
| `npm run typecheck` | `tsc --noEmit` |
| `npm run setup:cloudflare` | Cloudflare 리소스 자동 생성 |

### 6-4. 설치 후 사용

1. 대시보드 접속 → GitHub OAuth 로그인
2. LLM 제공자 등록 (API 키는 암호화되어 DB에 저장)
3. 저장소별 설정: 트리거 이벤트, 스킵 glob, draft 처리, 라벨, 커스텀 룰, 리뷰 예산, 모델 체인
4. PR 생성 시 자동 리뷰 / 또는 PR에서 봇 멘션으로 수동 트리거

### 6-5. 텔레메트리 주의

`.dev.vars.example` 기준 **익명 집계 통계가 기본 활성화**되어 있습니다.
`https://codra.run/api/telemetry` 로 전송되며, 끄려면:

```bash
TELEMETRY_DISABLED="true"
```

---

## 7. Q&A 정리

### Q1. 플러그인인가, 스킬인가, MCP인가?

**셋 다 아닙니다.** 독립 배포되는 **풀스택 웹 애플리케이션 + GitHub App**입니다.

| 종류 | 실행 위치 | 호출 주체 | Codra 해당 |
|---|---|---|---|
| 플러그인 | 호스트 앱 내부 | 호스트 앱 | ❌ |
| 스킬 | AI 에이전트 문맥 | AI 모델 | ❌ |
| MCP 서버 | 별도 프로세스 (MCP 프로토콜) | AI 클라이언트 | ❌ |
| **Codra** | **내 Cloudflare 계정** | **GitHub 웹훅** | ✅ |

**근거:** MCP 의존성이 전혀 없고, `apps/worker/wrangler.jsonc`에 Cloudflare 바인딩(KV, Queues, Hyperdrive, AI)이 정의되어 있으며, 진입점이 `fetch` / `queue` / `scheduled` 핸들러입니다.

> 참고: `packages/models`가 이미 여러 LLM 제공자를 추상화해 두었기 때문에,
> 그 위에 MCP 어댑터를 얹어 **"Codra를 MCP 서버로 감싸는 것"** 은 충분히 가능합니다.

### Q2. API 토큰이 필요한가?

**필요합니다. 4종류입니다.**

| # | 자격증명 | 환경변수 | 비용 |
|---|---|---|---|
| 1 | GitHub App | `GITHUB_APP_ID`, `APP_PRIVATE_KEY`, `GITHUB_APP_WEBHOOK_SECRET`, `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET` | 무료 |
| 2 | LLM API 키 | 대시보드에서 등록 (`LLM_CONFIG_ENCRYPTION_KEY`로 암호화) | **사용량 과금** |
| 3 | Cloudflare API 토큰 | `CF_ACCOUNT_ID`, `CF_API_TOKEN` (Workers AI Read 권한) | 무료 |
| 4 | PostgreSQL | `DATABASE_URL`, Hyperdrive 연결 문자열 | 무료 티어 가능 |

#### 비용 감각 (월 PR 100개 · PR당 파일 10개 가정)

| 등급 | 모델 예시 | 월 예상 |
|---|---|---|
| 경제형 | Gemini Flash / 소형 모델 | $3 ~ $10 |
| 균형형 | 중급 모델 | $15 ~ $40 |
| 최상형 | 최상위 모델 | $80 ~ $200 |
| 인프라 | Cloudflare Workers 유료 + PG | $5 ~ $25 |

> **비용 절감 팁:** Codra의 **모델 체인 + 크기 기반 오버라이드**를 활용해
> 작은 PR은 저렴한 모델, 큰 PR만 고급 모델로 라우팅하면 비용이 크게 줄어듭니다.

### Q3. 왜 GitHub에서 주목받는가?

> 스타 수는 이 문서 작성 환경에서 직접 확인하지 않았습니다. 아래는 코드 근거 기반의 "인기 요인" 분석입니다.

1. **타이밍** — AI 코드 리뷰가 가장 뜨거운 분야인데 오픈소스 대안이 드물었음
2. **코드 유출 우려 해결** — 셀프호스팅으로 기업의 1순위 도입 장벽 제거
3. **Cloudflare 네이티브** — Docker/K8s/EC2 없이 `npm run deploy` 한 번
4. **교과서급 아키텍처** — 단방향 의존성, 플랫폼 무관 core, ports/adapters
5. **벤더 락인 없음** — 7개 LLM 제공자 지원
6. **살아있는 디테일** — 예산 관리, KV 최적화, job 복구, verify 게이트 + "왜 그렇게 했는지" 주석
7. **프로급 운영** — 전용 사이트, changesets 릴리스, CLA, SECURITY.md, Playwright 테스트

#### 냉정하게 볼 점

- 버전 `0.9.5` **베타** — README에 breaking changes 예고
- **Cloudflare 종속** — 이전하려면 어댑터 재작성 필요
- **초기 설정 장벽** — 5개 부품 조립 필요
- **AGPL + CLA** — 상업적 활용에 제약

### Q4. 로컬 에이전트 구축에 도움이 되는가?

**매우 도움이 됩니다.** 에이전트 개발에서 막히는 지점의 해답이 대부분 들어 있습니다.

| 에이전트 개발 고민 | Codra의 해답 | 파일 |
|---|---|---|
| 모델 교체 추상화 | 포트 인터페이스 + 어댑터 | `packages/core/src/ports/model.ts` |
| 모델 실패 대응 | 체인 + 폴백 러너 | `packages/models/src/internal/model-chain-runner.ts` |
| 깨진 JSON 응답 | `jsonrepair` + 관대한 파서 | `packages/core/src/model-output/json.ts` |
| AI 헛소리 제거 | 증거 매칭 + 재검증 게이트 | `packages/core/src/finding-gates.ts`, `verify.ts` |
| 중복 응답 | dedupe 모듈 | `packages/core/src/model-output/dedupe.ts` |
| 컨텍스트 초과 | 파일 단위 배치 + pack | `packages/core/src/review/pack.ts`, `bin-runner.ts` |
| 비용 폭주 | 예산 기반 동적 제한 | `packages/core/src/review/budget.ts` |
| 장시간 작업 | 큐 + 워크플로 + lease 복구 | `apps/worker/src/workflows/review.ts` |
| 프롬프트 관리 | 용도별 분리 | `packages/core/src/prompts/*` |
| 재시도 정책 | 전용 모듈 | `packages/core/src/review/retry-policy.ts` |

#### 로컬 에이전트로 전환하는 방법

`core`가 플랫폼 무관하게 설계되어 있어 **포트만 교체하면** 됩니다.

| Cloudflare | 로컬 대체 |
|---|---|
| Cloudflare Queues | BullMQ / Redis / 인메모리 큐 |
| Cloudflare KV | Redis / SQLite / Map |
| Hyperdrive | PostgreSQL 직접 연결 |
| Workers Runtime | Node.js + Hono (Hono는 Node 지원) |

> `packages/core/src/ports/in-memory.ts` 에 **인메모리 구현체가 이미 존재**합니다.
> 테스트용으로 만들어진 것이지만 로컬 에이전트의 출발점으로 그대로 쓸 수 있습니다.

#### 추천 학습 순서

```text
1. packages/schema           도메인 모델 파악
2. packages/core/src/ports   추상화 설계 철학
3. packages/core/src/prompts 프롬프트 엔지니어링
4. packages/core/src/review  파이프라인 오케스트레이션
5. packages/core/src/finding-gates.ts  품질 제어 (최고 가치)
6. packages/models/src/internal/*      모델 체인 / 폴백
```

### Q5. React나 PHP로 만들 수 있는가?

#### React — 이미 React입니다

`apps/dashboard`가 통째로 **React 19 + Vite 8 + Tailwind 4**입니다.

```text
apps/dashboard/src/
├── pages/       dashboard, jobs, job-detail, repos, stats, settings, account, login, landing
├── hooks/       use-session, use-polling, use-job-detail, use-review-settings, use-can 등
├── lib/         api, diffs-cache, job-format, batch-groups, timezone
└── components/  shared / features / layout (3계층 분리)
```

→ 한국어화, 디자인 변경, 페이지 추가 모두 바로 가능합니다.

#### PHP — 재구현 가능 (Laravel 권장)

| Codra (현재) | PHP / Laravel 대체 |
|---|---|
| Cloudflare Workers | PHP-FPM + Nginx / Laravel Octane |
| Hono 라우터 | Laravel Routes / Controllers |
| Cloudflare Queues | Laravel Queue (Redis / DB 드라이버) |
| Cloudflare Workflows | Laravel Job Batches / Chains |
| Cloudflare KV | Redis / Laravel Cache |
| Hyperdrive + PG | PDO / Eloquent |
| Zod 스키마 | Laravel Validation / `spatie/laravel-data` |
| 웹훅 HMAC 검증 | `hash_hmac('sha256', ...)` + `hash_equals()` |
| GitHub App JWT | `firebase/php-jwt` |
| LLM 호출 | Guzzle HTTP / `openai-php/client` |

**PHP에서 오히려 쉬워지는 점**

- 서브리퀘스트 제한 없음 → `budget.ts` 수준의 복잡한 예산 로직 불필요
- CPU 시간 제한 없음 → 긴 리뷰를 한 번에 처리
- 파일 시스템 자유롭게 사용
- 저렴한 호스팅에서도 동작

**PHP에서 주의할 점**

- 웹훅은 **즉시 202 응답 후 큐 적재** (GitHub 웹훅 타임아웃 약 10초)
- `php artisan queue:work` 데몬을 Supervisor 등으로 관리 필요
- 콜드스타트는 없지만 서버 운영 부담 발생

**권장 조합**

```text
프론트: React (Codra 대시보드 참고)
백엔드: Laravel + PostgreSQL + Redis Queue
AI:     core의 프롬프트 / 게이트 "설계"를 PHP로 이식

핵심: 코드를 복사하지 말고 설계 패턴만 참고 → AGPL 제약에서 자유로움
```

**예상 난이도:** 1인 MVP 기준 약 3~6주.
어려운 부분은 인프라가 아니라 **"AI가 오탐을 내지 않게 만드는 것"** 입니다.

---

## 8. 수익화 아이디어

### 8-0. 반드시 먼저 알아야 할 라이선스 이슈

#### AGPL-3.0의 네트워크 조항

| 라이선스 | 소스 공개 의무 발생 시점 |
|---|---|
| GPL | 소프트웨어를 **배포**할 때 |
| **AGPL** | 소프트웨어를 **네트워크 서비스로 제공**할 때도 |

> Codra를 수정해서 웹서비스로 제공하면, **수정한 소스코드를 이용자에게 공개**해야 합니다.
> "우리 서버에서만 돌리니 괜찮다"는 논리는 AGPL에서 통하지 않습니다.

#### CLA가 시사하는 것

`CONTRIBUTING.md`에 다음 문구가 있습니다.

> "our **dual-licensing model** (AGPL-3.0 for the core)"

즉 원작자는 **AGPL(무료) + 상용 라이선스(유료)** 투트랙을 운영/준비 중이며,
**상용 라이선스를 구매하는 경로가 존재**한다는 뜻입니다.

#### 3가지 전략

| 전략 | 리스크 | 설명 |
|---|---|---|
| 🟢 A. AGPL 준수 | 낮음 | 수정본을 공개하고 **서비스/운영**으로 수익 |
| 🟡 B. 상용 라이선스 구매 | 중간 | 원작자와 협상해 비공개 권리 확보 |
| 🔵 C. 패턴만 참고해 신규 개발 | 없음 | 코드 복사 없이 아키텍처 아이디어만 활용 |

> ⚖️ 실제 사업화 전에는 **오픈소스 라이선스 전문 변호사 상담이 필수**입니다.
> 본 문서는 참고용이며 법률 자문이 아닙니다.

---

### 8-1. 전략 A — AGPL 범위 안에서 수익화

AGPL은 **코드 판매**를 제약하지만 **서비스 판매**는 막지 않습니다.

#### A-1. 한국형 매니지드 호스팅

```text
문제: 설치가 복잡 (Cloudflare + PG + GitHub App + LLM 키)
해결: 가입 후 저장소 연결만 하면 끝

가격안: 스타터 3만원/월 · 팀 12만원/월 · 엔터프라이즈 별도
차별점: 한국어 리뷰 코멘트 / 카카오톡·슬랙 알림 / 국내 결제·세금계산서 / 한국어 CS
준수:   내 포크를 공개 저장소로 유지 (AGPL 충족)
```

코드가 공개되어도 **운영·가동시간·지원·편의성**은 복제되지 않습니다.
Red Hat, GitLab, Grafana가 쓰는 모델입니다.

#### A-2. 엔터프라이즈 구축 + 유지보수 ★ 추천 1순위

```text
타깃: 금융 / 공공 / 의료 등 코드 외부 반출이 금지된 조직

상품 구성:
  구축 컨설팅            500만 ~ 2,000만원 (1회)
  연간 유지보수           300만 ~ 800만원/년
  사내 코딩 컨벤션 룰셋     200만 ~ 500만원
  개발자 교육             건당 100만 ~ 300만원

AGPL 이슈: 없음 (서비스 제공이지 소프트웨어 재배포가 아님)
```

초기 자본 거의 0, 마진 최고, 법적 리스크 최소.

#### A-3. 커스텀 룰셋 마켓플레이스

Codra는 저장소별 **custom rules** 설정을 지원합니다. 이는 코드가 아니라 **설정 데이터**입니다.

```text
금융권 보안 규정 팩          월 5만원
K-접근성 / 개인정보보호법 팩   월 3만원
React 성능 안티패턴 팩        월 3만원
Laravel 베스트프랙티스 팩     월 3만원
의료 데이터 처리 규정 팩      월 8만원

AGPL 이슈: 없음 (설정 데이터는 파생저작물이 아님)
```

#### A-4. 분석 레이어를 별도 서비스로

Codra는 "리뷰"만 합니다. 그 위에 독립 SaaS를 얹습니다.

```text
· 팀 코드 품질 추이 대시보드
· 개발자별 성장 리포트
· 반복되는 버그 패턴 분석
· LLM 비용 최적화 리포트
· 기술부채 히트맵

Codra API를 소비하는 독립 제품 → AGPL 전염 없음
가격: 팀당 월 5만 ~ 30만원
```

#### A-5. 통합 브릿지 서비스

```text
Codra ↔ Jira / Notion / 카카오워크 / 슬랙 / 두레이

예) P1 버그 발견 시 Jira 티켓 자동 생성
예) 주간 리뷰 요약을 Notion에 자동 정리

별도 프로세스로 구현 → AGPL 무관
가격: 월 3만 ~ 10만원
```

#### A-6. 콘텐츠 & 교육 (리스크 0)

```text
· 유튜브: "AI 코드리뷰 봇 직접 만들기" 시리즈
· 전자책: "Cloudflare Workers로 AI 에이전트 만들기" (3~5만원)
· 온라인 강의 (15~30만원)
· 기술블로그 → 컨설팅 리드 확보 (A-2로 연결)
```

---

### 8-2. 전략 B — 상용 라이선스 협상

```text
대상: Devarshi Shimpi (원작자)
경로: codra.run 을 통해 dual-license 상용 계약 문의

확보 가능한 권리:
  · 수정본 비공개
  · 자체 브랜딩 SaaS 운영
  · 독점 기능 개발

예상 비용: 연 수백만 ~ 수천만원 (협상)
적합 시점: 매출이 검증된 이후 단계
```

---

### 8-3. 전략 C — 패턴만 참고해 새로 개발 (가장 자유로움)

```text
✅ 합법: 아키텍처 아이디어, 설계 패턴, 문제 해결 접근법
❌ 불법: 소스코드 복사, 프롬프트 텍스트 복붙, 파생 수정 후 비공개 서비스

참고할 아이디어:
  · 포트 / 어댑터로 LLM 추상화
  · 웹훅 → 큐 → 워커 비동기 파이프라인
  · 발견사항 재검증으로 오탐 제거 ★
  · 모델 체인 + 폴백
  · 비용 예산 기반 작업량 제어

직접 만들 것:
  Laravel 또는 Node 백엔드 + React 프론트
  한국 시장 특화 (한국어 리뷰, 국내 결제, 국내 협업툴 연동)
  → 100% 자체 IP, 라이선스 자유, 투자 유치 가능
```

---

### 8-4. 한국 시장이 기회인 이유

1. **한국어 리뷰 코멘트** — 글로벌 서비스가 잘 대응하지 못하는 영역
2. **국내 결제 / 세금계산서** — 기업 구매 프로세스의 필수 조건
3. **공공·금융 망분리 규제** — 해외 SaaS 원천 차단 → 셀프호스팅이 유일한 답
4. **카카오워크 / 두레이 / 잔디** — 국내 협업툴 연동은 국내 업체만 수행
5. **한국어 기술지원** — 엔터프라이즈 계약의 결정적 요소

---

### 8-5. 실행 로드맵

| 기간 | 목표 | 활동 | 예상 매출 |
|---|---|---|---|
| 1개월차 | 검증 | 직접 설치·운영, 오탐률 실측, 한국어 프롬프트 품질 비교, 비용 실측 | 0원 |
| 2~3개월차 | 첫 매출 | 지인 회사 1곳 무료 구축 → 레퍼런스 확보, 콘텐츠 3~5편, 첫 유료 구축 계약 | 500만~1,000만원 |
| 4~6개월차 | 제품화 | 한국어 룰셋 팩(A-3), 분석 대시보드 MVP(A-4), 호스팅 베타(A-1) | 월 100만~300만원 |
| 7~12개월차 | 확장 | 전략 C 자체 제품 착수, 엔터프라이즈 3~5곳 확보 | 월 500만~1,500만원 |

---

### 8-6. 최종 추천 순위

| 순위 | 전략 | 이유 |
|---|---|---|
| 🥇 | **A-2 엔터프라이즈 구축 / 컨설팅** | 초기 자본 0, 마진 최고, 법적 리스크 최소, 즉시 시작 가능 |
| 🥈 | **A-3 커스텀 룰셋 판매** | 반복 매출, AGPL 무관, 지식의 자산화 |
| 🥉 | **C 자체 제품 개발** | React/PHP 역량 기반 장기 승부, 100% 자체 IP |

#### 피해야 할 선택

처음부터 **A-1 매니지드 호스팅 SaaS**로 시작하는 것.
인프라 운영 + 24/7 고객 지원 + AGPL 공개 의무 + 원작자와의 직접 경쟁이 겹쳐 난이도가 가장 높습니다.

> 권장 경로: **A-2로 현금 흐름 확보 → A-3 / A-4로 확장 → C로 독립**

---

## 9. 핵심 요약

| 질문 | 답 |
|---|---|
| 이게 뭐야? | 셀프호스팅 AI 코드 리뷰 엔진 (GitHub App + Cloudflare Worker + React 대시보드) |
| 언제 써? | 코드 외부 반출이 어렵거나, SaaS 구독 대신 직접 소유하고 싶을 때 |
| 플러그인/스킬/MCP? | 전부 아님. 독립 웹 애플리케이션 |
| 토큰 필요? | GitHub App + LLM API 키 + Cloudflare 토큰 + PostgreSQL |
| 학습 가치? | 매우 높음 — LLM 에이전트 설계의 실전 교본 |
| React/PHP 가능? | React는 이미 사용 중, PHP(Laravel) 재구현도 충분히 가능 |
| 수익화? | 가능하되 **AGPL-3.0 제약**이 핵심 변수 (컨설팅/룰셋이 가장 안전) |

---

*이 문서는 저장소 소스를 직접 분석한 내용을 기반으로 작성되었습니다.*
*라이선스 및 사업화 관련 내용은 참고용이며, 실제 사업화 전 전문가 검토가 필요합니다.*
