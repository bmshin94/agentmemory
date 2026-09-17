# agentmemory 분석 정리 (한국어)

> AI 코딩 에이전트용 영속 메모리 도구 `agentmemory`를 분석하고,
> 활용 방안과 수익화 가능성까지 검토한 문서입니다.

**작성일:** 2026-09-17
**분석 대상 버전:** `v0.9.29`

## 🔗 관련 링크

| 구분 | 주소 |
|---|---|
| 원본 저장소 | https://github.com/rohitg00/agentmemory |
| 내 포크 | https://github.com/bmshin94/agentmemory |
| npm 패키지 | https://www.npmjs.com/package/@agentmemory/agentmemory |
| 기반 엔진 (iii) | https://github.com/iii-hq/iii |
| 설계 원본 Gist (⭐1.6k) | https://gist.github.com/rohitg00/2067ab416f7bbe447c1977edaaa681e2 |
| 한국어 README | https://github.com/rohitg00/agentmemory/blob/main/READMEs/README.ko-KR.md |
| 에이전트용 설치 문서 | https://raw.githubusercontent.com/rohitg00/agentmemory/main/INSTALL_FOR_AGENTS.md |

---

## 1. 이게 뭐하는 물건인가

한 줄 요약: **AI 코딩 에이전트에게 장기기억을 붙여주는 도구.**

> *"Your coding agent remembers everything. No more re-explaining."*

### 해결하는 문제

Claude Code, Cursor, Copilot 같은 도구는 세션이 끝나면 맥락을 전부 잃는다.
그래서 매번 다음을 반복 설명해야 한다.

- 프로젝트 구조
- 손대면 안 되는 영역
- 지난번에 이미 지적했던 실수

agentmemory는 로컬에 메모리 서버를 띄워두고,
에이전트의 작업을 **자동으로 기록 → 압축 → 다음 세션에 주입**한다.

### 지원 에이전트 (20종 어댑터)

Claude Code, Cursor, GitHub Copilot CLI, Gemini CLI, Codex CLI,
OpenCode, Devin, Droid(Factory.ai), Antigravity, Hermes, OpenClaw, pi 등.
MCP를 지원하는 클라이언트라면 대부분 연결 가능하다.

---

## 2. 아키텍처 — 플러그인? 스킬? MCP?

**전부 다 맞다.** 4개 층으로 쌓인 구조다.

```
┌─────────────────────────────────────────────┐
│  🎓 스킬 17개        AI에게 "언제 쓰는지" 교육   │
├─────────────────────────────────────────────┤
│  🪝 플러그인/훅 12개   작업 내용 자동 캡처        │
├─────────────────────────────────────────────┤
│  🔌 MCP 서버 (툴 54개) AI가 호출하는 창구        │
├─────────────────────────────────────────────┤
│  🧠 메모리 서버 + REST API (localhost:3111)    │
└─────────────────────────────────────────────┘
```

| 층 | 역할 | 비유 |
|---|---|---|
| REST 서버 | 본체. 100개 이상 엔드포인트 | 본사 창고 |
| MCP 서버 | 표준 프로토콜 창구 | 주문 전화 |
| 플러그인(훅) | 자동 기록 | CCTV |
| 스킬 | 사용 시점 안내 | 매뉴얼 |

### 주요 디렉터리

| 경로 | 내용 |
|---|---|
| `src/` | 코어 (MCP, 훅, CLI, REST, 프로바이더, 뷰어) |
| `plugin/skills/` | 스킬 17개 (`remember`, `recall`, `forget`, `lesson`, `handoff`, `recap`, `commit-context` 등) |
| `plugin/hooks/` | 에이전트별 훅 설정 6종 (claude / copilot / codex / droid / devin / antigravity) |
| `plugin/scripts/` | 실제 훅 실행 스크립트 (`.mjs`) |
| `src/viewer/` | 실시간 뷰어 — 바닐라 HTML **4,572줄** |
| `website/` | 공식 사이트 — **Next.js 16 + React 19** |
| `integrations/` | 커넥터 (filesystem-watcher, hermes, pi, openclaw) |
| `packages/mcp/` | 독립 배포되는 MCP shim |
| `deploy/` | Fly / Render / Railway / Coolify 배포 설정 |
| `benchmark/`, `eval/` | 10만건 부하 테스트, LongMemEval 평가 |

### ⚠️ MCP shim 함정

`@agentmemory/mcp`만 단독 설치하면 **툴 7개짜리 반쪽**이다.
서버(`localhost:3111`)가 떠 있어야 **54개 풀 세트**가 열린다.
Cursor에서 툴이 7개만 보이면 서버가 꺼진 것이다.

---

## 3. 동작 방식

### 메모리 파이프라인

```
PostToolUse 훅 발동
  → SHA-256 중복 제거 (5분 윈도우)
  → 프라이버시 필터 (시크릿/API키 제거)
  → 원본 관찰 저장
  → 압축 (기본은 합성 압축, LLM 압축은 옵션)
  → 임베딩 (프로바이더 활성 시)
  → BM25 + 벡터 색인

Stop / SessionEnd 훅
  → 세션 요약 → 지식그래프 추출 → 슬롯 성찰

SessionStart 훅
  → 프로젝트 프로필 로드
  → 하이브리드 검색 (BM25 + 벡터 + 그래프)
  → 토큰 예산 적용 (기본 2000) → 대화에 주입
```

### 4계층 기억 구조 (인간 뇌 모델링)

| 계층 | 내용 | 비유 |
|---|---|---|
| Working | 원본 관찰 로그 | 단기 기억 |
| Episodic | 압축된 세션 요약 | "무슨 일이 있었나" |
| Semantic | 추출된 사실/패턴 | "내가 아는 것" |
| Procedural | 워크플로우/의사결정 | "이건 이렇게 한다" |

에빙하우스 망각곡선을 적용해 **자주 쓰는 기억은 강화되고,
안 쓰는 기억은 자동 퇴출**된다. 모순되는 정보는 감지 후 정리한다.

### 캡처 시점 (훅 12종)

| 훅 | 캡처 내용 |
|---|---|
| `SessionStart` | 프로젝트 경로, 세션 ID |
| `UserPromptSubmit` | 사용자 프롬프트 (필터링 후) |
| `PreToolUse` | 파일 접근 패턴 |
| `PostToolUse` | 툴 이름, 입력, 출력 |
| `PostToolUseFailure` | **에러 맥락** |
| `PreCompact` | 컨텍스트 압축 전 기억 재주입 |
| `SubagentStart/Stop` | 서브에이전트 생명주기 |
| `Stop` / `SessionEnd` | 세션 요약 / 종료 마커 |

---

## 4. 설치 및 사용법

### 요구사항

- Node.js 20 이상
- macOS/Linux: `curl`, `sh`, `tar`
- Windows: **WSL2 권장** (네이티브는 `iii.exe` 수동 설치 필요)

### 설치

```bash
npx -y @agentmemory/agentmemory@latest
```

첫 실행 시 대화형 마법사가 에이전트 선택, LLM 프로바이더 선택,
전역 설치 여부를 물어보고 설정/서버/엔진을 자동 구성한다.

### 동작 확인

```bash
npx -y @agentmemory/agentmemory@latest demo   # 샘플 세션 3개 + 검색 테스트
npx skills add rohitg00/agentmemory -y        # 스킬 17개 설치
open http://localhost:3113                    # 실시간 뷰어
```

### 일상 명령어

| 명령어 | 기능 |
|---|---|
| `agentmemory` | 서버 시작 |
| `agentmemory stop` | 정상 종료 |
| `agentmemory connect <agent>` | 에이전트 추가 연결 |
| `agentmemory doctor` | 진단 + 수리 |
| `agentmemory status` | 상태 확인 |
| `agentmemory remove` | 전체 제거 |

### 포트

| 포트 | 용도 |
|---|---|
| 3111 | REST API + MCP |
| 3112 | iii 스트림 |
| 3113 | **뷰어** |
| 49134 | iii 워커 WebSocket |

Docker도 지원한다 (`docker-compose.yml`).

---

## 5. API 토큰이 필요한가

**필요 없다.** `.env.example` 첫머리에 명시되어 있다.

> *"Every line is OFF by default — agentmemory runs out of the box with
> no LLM key, no embedding key, and no API auth."*

| 기능 | 키 없음 (무료) | 키 있음 |
|---|---|---|
| 자동 기록 | ✅ | ✅ |
| 키워드 검색 (BM25) | ✅ | ✅ |
| 의미 검색 (벡터) | ❌ | ✅ |
| 세션 요약 / 성찰 | ❌ | ✅ |
| LLM 압축 | 합성 압축 | 실제 요약 |

### 💡 무료로 의미검색 켜기

```env
# ~/.agentmemory/.env
EMBEDDING_PROVIDER=local
```

`Xenova/all-MiniLM-L6-v2` (384차원)를 최초 1회만 내려받고,
이후에는 **온디바이스**로 추론한다. 비용 0원.

### 지원 프로바이더 (감지 우선순위)

```
OPENAI_API_KEY → MINIMAX_API_KEY → ANTHROPIC_API_KEY
→ GEMINI_API_KEY → OPENROUTER_API_KEY → noop
```

임베딩 전용: `VOYAGE_API_KEY`(코드 특화), `COHERE_API_KEY`.
`AGENTMEMORY_SECRET`은 API 키가 아니라 **REST 노출 시 쓰는 자체 비밀번호**다.

> 추천 시작 세팅: **키 없이 + `EMBEDDING_PROVIDER=local`**

---

## 6. 왜 GitHub에서 유명한가

1. **출발점** — Karpathy의 "LLM Wiki" 패턴을 확장한 설계 Gist가 ⭐1.6k / 포크 230개를 먼저 달성. 팬덤을 만들고 시작했다.
2. **보편적 페인포인트** — "AI가 매번 까먹는다"는 모든 사용자의 불만.
3. **수치 제시** — 검색 R@5 95.2%, 토큰 92% 절감, MCP 툴 54개, 자동 훅 12개, 외부 DB 0개, 테스트 1,674개.
4. **넓은 호환성** — 20개 에이전트 어댑터.
5. **타이밍** — MCP 생태계 확산기와 정확히 맞물림.
6. **문서량** — README 약 10만 자, 12개 언어 번역, 설계문서/로드맵/거버넌스/벤치마크 문서 완비. AI에게 링크만 주면 알아서 설치하는 문서까지 존재.
7. **로컬 우선** — 데이터가 외부로 안 나가서 기업 도입 장벽이 낮다.

### 경쟁 제품

| 제품 | 스타 | 검색 R@5 | 캡처 방식 |
|---|---|---|---|
| **agentmemory** | — | **95.2%** | 훅 12개 (자동) |
| mem0 | 63K | 68.5% | 수동 `add()` |
| MemPalace | 54K | ~96.6% (자체) | 수동 |
| Khoj | 36K | N/A | 수동 |
| supermemory | 29K | 자체 보고 | API 추출 |
| Letta / MemGPT | 24K | 83.2% | 에이전트 자체 편집 |

---

## 7. 로컬 에이전트 구축에 도움이 되는가

**된다.** 세 가지 방식.

### A. 그대로 갖다 쓰기 (가장 쉬움)

```bash
# 저장
curl -X POST http://localhost:3111/agentmemory/remember \
  -H 'Content-Type: application/json' \
  -d '{"content":"이 프로젝트는 PHP 8.3 사용","concepts":["stack"]}'

# 검색
curl -X POST http://localhost:3111/agentmemory/smart-search \
  -H 'Content-Type: application/json' \
  -d '{"query":"PHP 버전","limit":5}'
```

HTTP만 되면 어떤 언어에서도 붙일 수 있다.

### B. 교과서로 활용

- 훅으로 에이전트 생명주기를 가로채는 법
- BM25 + 벡터 + 그래프를 **RRF로 융합**하는 검색 기법
- 4계층 기억 + 망각곡선 설계
- MCP 서버 구현 (`src/mcp/`)
- 20개 에이전트 어댑터 패턴 (`src/cli/connect/`)

### C. 완전 오프라인 구성

```env
OPENAI_BASE_URL=http://localhost:11434/v1   # Ollama
OPENAI_API_KEY=ollama
EMBEDDING_PROVIDER=local
```

OpenAI 호환 엔드포인트를 지원하므로 Ollama / LM Studio / vLLM 연결 가능.
**인터넷 없이 100% 로컬에서 도는 기억형 에이전트**를 만들 수 있다.

---

## 8. React / PHP로 만들 수 있는가

### 현재 스택

| 부분 | 언어 |
|---|---|
| 코어 서버 | TypeScript / Node.js |
| 저장 엔진 | iii-engine (별도 바이너리, v0.11.2 고정) |
| 뷰어 | 바닐라 HTML 4,572줄 |
| 공식 사이트 | Next.js 16 + React 19 |
| 훅 스크립트 | Node `.mjs` |

### ⚛️ React

- **프론트엔드는 100% 가능하며, 가장 좋은 진입점이다.**
- REST API가 이미 전부 열려 있어 **백엔드 수정 없이 UI만** 새로 만들 수 있다.
- 현재 뷰어가 바닐라 HTML 4,572줄이라 개선 여지가 크다.
- 만들 만한 것: 기억 대시보드, 지식그래프 시각화(`react-force-graph`), 세션 리플레이 플레이어, 모바일 앱(React Native).
- 단, 코어 서버를 React로 대체하는 것은 불가능(역할이 다름).

### 🐘 PHP

- **API 연동은 매우 쉽다** (cURL 한 번). Laravel / WordPress에 기억 기능 붙이기 용이.
- 훅 스크립트를 PHP로 바꾸는 것도 기술적으로 가능하나 실익이 적다.
- **전체 재구현은 비추천.** 특히:

| 기능 | PHP 대안 | 난이도 |
|---|---|---|
| REST 서버 | Laravel / Slim | 🟢 |
| BM25 | MySQL 전문검색 / Meilisearch | 🟡 |
| 벡터 검색 | PostgreSQL + pgvector | 🟡 |
| 지식그래프 | Neo4j / 관계형 테이블 | 🟠 |
| **MCP 서버** | **PHP SDK 미성숙** | 🔴 |
| 로컬 임베딩 | PHP에 없음 (Python 호출 필요) | 🔴 |

### 권장 조합

```
🎨 프론트: React / Next.js         ← 새로 만든다 (차별화 지점)
          ↓ REST 호출
🧠 백엔드: agentmemory 그대로 사용   ← 건드리지 않는다
          ↓ (선택)
🐘 PHP: 사내 레거시 연동 레이어
```

---

## 9. 수익화 검토

### 🚨 핵심 발견: 상업화 트랙이 비어 있다

`ROADMAP.md`의 **"Out of scope"** 섹션에 명시되어 있다.

> - ❌ A cloud-hosted agentmemory SaaS
> - ❌ Billing, subscription tiers, commercial licensing beyond Apache-2.0

**원작자가 상업화를 하지 않겠다고 공개 선언**한 것이다.
오픈소스 사업의 최대 리스크(원작자가 같은 걸 공식 출시)가 문서로 차단되어 있다.

### ⚖️ 라이선스: Apache-2.0

| 가능 | 내용 |
|---|---|
| ✅ | 상업적 이용 / 판매 |
| ✅ | 수정 후 **비공개 유지** (GPL과 다름) |
| ✅ | SaaS 제공 (AGPL이 아님) |
| ✅ | 특허 라이선스 포함 |

**준수 사항**
1. LICENSE 사본 포함
2. 저작권 고지(NOTICE) 유지
3. 수정 파일에 수정 사실 표시

**금지 사항**
- "agentmemory" 상표 그대로 사용 ❌ (상표는 라이선스 대상이 아님)
  - ⭕ `MemoryFlow (powered by agentmemory)`
  - ❌ `agentmemory Korea`, `agentmemory Pro`
- 원작자의 보증/후원인 것처럼 표시 ❌

> 실제 사업화 전 IT 전문 변호사 상담 필수.

### 🚫 피해야 할 영역

로드맵 **Q4 2026 (Trust)** 에 무료로 예정된 기능들:

```
□ SSO 게이트웨이 (OIDC)     □ 감사 로그 내보내기
□ RBAC 권한 관리            □ Docker/systemd 배포 가이드
□ 보안 감사
```

→ 이것만으로 사업을 만들면 1년 뒤 무료화된다.
또한 `deploy/`에 Fly / Render / Railway / Coolify 설정이 이미 있어
**단순 관리형 호스팅은 차별화가 약하다.**

### 💡 수익화 원리

```
코드는 공짜다 → 코드를 팔면 안 된다.

팔아야 하는 것:
  ⏰ 시간 (설치·운영을 대신)
  🛡️ 책임 (문제 발생 시 책임질 주체)
  🎯 맥락 (우리 업종/회사 전용)
  🤝 신뢰 (연락되는 사람, 계약서, SLA)
```

### 아이디어 7선

#### 🥇 1. 한국형 온프렘 구축 + 상용 지원(SLA) — 최우선 추천

금융/공공/대기업 **폐쇄망**에 구축하고 연간 유지보수 계약을 맺는 모델.

**근거**
- 국내 금융·공공은 규정상 외부 클라우드 사용 불가 → mem0, supermemory 등 **해외 클라우드 SaaS 경쟁자는 입찰 자격 자체가 없다.**
- 프로젝트 미션이 *"Requires zero external databases"*, *"Keeps every user's data on the user's machine by default"* → **폐쇄망에 최적화된 설계.**
- **`MAINTAINERS.md` 확인 결과 메인테이너가 1명(Rohit Ghumare, Independent), Emeritus 없음 → 버스 팩터 = 1.**
  기업 구매팀이 가장 우려하는 지점이며, **바로 이 불안을 해소해주는 것이 판매 상품**이 된다.
  (Red Hat이 이 모델로 성장했다.)

**가격 구조 (국내 기준)**

| 항목 | 금액 |
|---|---|
| 초기 PoC | 500만 ~ 1,500만원 |
| 본 구축 (커스터마이징 포함) | 2,000만 ~ 5,000만원 |
| **연간 유지보수** | **구축비의 15~20% / 년** |
| 긴급 SLA (4시간 대응) | 별도 +30% |

**추가 개발 필요**
- 한국어 형태소 분석 (`@node-rs/jieba`가 중국어용으로 이미 있음 → mecab 연동)
- 오프라인 설치 패키지
- 한국어 관리자 매뉴얼
- 보안 가이드라인 대응 문서

**리스크** — 영업 사이클이 길다(6개월~1년). 첫 레퍼런스 확보가 관건.

#### 🥈 2. 팀 지식 대시보드 SaaS

개인 기억은 무료 OSS가 잘한다. **팀 단위 가시화**가 비어 있다.

타깃 고통: *"담당자 퇴사 후 아무도 모름"*, *"온보딩 3개월"*, *"이 결정 왜 했는지 모름"*

| 화면 | 내용 |
|---|---|
| 팀 지식 그래프 | 누가 어떤 영역을 아는지 |
| **지식 편중 경고** | "결제 모듈은 1명만 안다" ← 킬러 기능 |
| 주간 리포트 | 팀이 배운 것 자동 요약 |
| 온보딩 팩 | 신입용 프로젝트 요약 생성 |
| 의사결정 추적 | 코드 → 그 결정을 한 세션으로 이동 |

REST API(`/agentmemory/graph/query`, `/agentmemory/team-feed` 등)가 이미 열려 있어
**백엔드 0줄, 프론트만 개발**하면 된다.

가격: Free(3명) / Team 1인 월 1.5만원 / Business 1인 월 3만원 + 온프렘 옵션

리스크: 로드맵 Q3 "Cross-agent shared memory namespace"가 **후보(Candidate)** 로 존재.
단 시각화/리포트는 로드맵에 없음.

#### 🥉 3. 한국 협업툴 커넥터 팩

`integrations/`에 커넥터 구조가 잡혀 있고, 공통 규격(`POST /agentmemory/observe`)으로 통일되어 있어 신규 제작이 쉽다.

로드맵에 있는 건 Slack / Discord / GitHub 뿐이다.

| 툴 | 로드맵 | 기회 |
|---|---|---|
| Jira / Confluence | ❌ | 대기업 필수 |
| Notion | ❌ | 스타트업 필수 |
| Dooray / 잔디 / 카카오워크 | ❌ | **경쟁자 없음** |
| 네이버웍스 | ❌ | **경쟁자 없음** |
| GitLab (설치형) | ❌ | 금융권 표준 |

가격: 커넥터당 월 3~5만원, 또는 커스텀 개발 건당 300~800만원

#### 4. 규제 산업 특화 (법률 / 의료 / 회계)

"AI가 무엇을 기억하는가"가 곧 컴플라이언스 문제인 업종.

- 🏥 병원: 환자정보 잔존 시 의료법 위반
- ⚖️ 로펌: 사건정보 혼입 시 이해충돌 사고
- 💼 회계: 감사 추적 필수

추가할 기능: 개인정보 자동 마스킹 강화, 고객사별 기억 격리 보증,
기억 삭제 법적 증빙 발급(잊힐 권리), 감사 대응 리포트.
기반은 이미 있다 (`memory_governance_delete`, `memory_audit`).

가격: 연 2,000만 ~ 1억원. 단 **도메인 전문가 파트너가 필요**하다.

#### 5. 교육 콘텐츠

"AI 에이전트 메모리" 주제의 한국어 자료가 사실상 없다.

| 상품 | 가격 |
|---|---|
| 온라인 강의 | 8~15만원 |
| 전자책 | 2~3만원 |
| 기업 출강 (1일) | 200~400만원/회 |
| 유튜브/블로그 | 무료 (마케팅) |

**진짜 목적은 매출이 아니라 1번·6번의 고객 유입 경로 확보.**

#### 6. 컨설팅 / 도입 SI — 가장 빠른 현금

"개발팀 AI 환경 구축" 안에 이 도구를 넣어 납품.

| 항목 | 금액 |
|---|---|
| 진단 리포트 (2주) | 300~500만원 |
| 구축 프로젝트 (2~3개월) | 1,500~4,000만원 |
| 상주 어드바이저 | 월 500~800만원 |

장점: 오늘 시작 가능, 현금흐름 즉시 발생, **고객 니즈 학습**(→ 1·2번 제품의 근거).
단점: 시간 투입에 비례해 확장 한계.

#### 7. 마이그레이션 도구

mem0 / Letta / supermemory 이탈 수요 대상.
무료 배포로 리드 수집하거나 건당 50~100만원.

### 📊 비교

| # | 아이디어 | 시작 속도 | 난이도 | 수익 규모 | 지속성 |
|---|---|---|---|---|---|
| 1 | 한국 온프렘 + SLA | 느림 | 높음 | 💰💰💰💰 | ⭐⭐⭐⭐⭐ |
| 2 | 팀 대시보드 SaaS | 보통 | 중간 | 💰💰💰 | ⭐⭐⭐⭐ |
| 3 | 한국 커넥터 | 빠름 | 낮음 | 💰💰 | ⭐⭐⭐ |
| 4 | 규제산업 특화 | 느림 | 높음 | 💰💰💰💰💰 | ⭐⭐⭐⭐⭐ |
| 5 | 교육 콘텐츠 | 빠름 | 낮음 | 💰 | ⭐⭐ |
| 6 | 컨설팅 | 즉시 | 낮음 | 💰💰💰 | ⭐⭐ |
| 7 | 마이그레이션 | 빠름 | 낮음 | 💰 | ⭐ |

### 🎯 실행 로드맵

**0~3개월 — 기반 확보 (매출 ≈ 0)**
- 직접 한 달 사용해보기
- 블로그/영상 3~5편 → 한국어 1등 콘텐츠 선점
- 한국어 검색 개선 PR을 원본에 기여 (전문성 증명 + 원작자와 관계 형성)
- 주변 개발팀 3곳 무료 설치 후 피드백 수집

**3~6개월 — 첫 매출 (월 300~1,000만원)**
- 컨설팅 1~2건 수주
- 커넥터 1개 완성 (Notion 또는 Jira)
- React 대시보드 MVP
- 온프렘 PoC 고객 1곳 확보 (할인해서라도 레퍼런스 확보)

**6~12개월 — 제품화 (월 2,000만원~)**
- 온프렘 정식 계약 2~3곳
- 대시보드 SaaS 유료 전환
- 브랜드 확립 및 법인화

### ⚠️ 리스크

1. **경쟁 강도** — mem0(63K⭐), MemPalace(54K⭐), Khoj(36K⭐) 등 투자받은 경쟁사 다수. 정면승부 대신 한국·폐쇄망·특화로 우회.
2. **원작자 정책 변경 가능성** — 대비책은 "제품"이 아니라 "관계와 신뢰"를 자산화하는 것.
3. **버전 불안정** — 아직 v0.9.29. API 프리즈는 **2027 Q1 예정**. 그전까지 커스터마이징 최소화.
4. **기술 트렌드** — 컨텍스트 무한 확장 시 필요성 감소 가능. 단 토큰 비용·보안 격리 때문에 당분간 유효.
5. **상표/라이선스 실수** — 가장 흔한 사고. 브랜드 신규 제작 + NOTICE 유지 + 법률 검토.

### 💬 결론

- 진입 우선순위: **6번(현금) → 5번(인지도) → 3번(제품 감각) → 1번(본 사업)**
- **제품을 먼저 만들지 말 것.** 사용 → 설치 지원 → 지불 의사 확인 → 그 다음 개발.
- 1번이 가장 유망한 이유는 기술력이 아니라 **"한국 기업 환경에 대한 이해"로 이기는 싸움**이기 때문이다. 해외 경쟁자가 넘어올 수 없는 해자다.

---

## 📌 한 장 요약

| 질문 | 답 |
|---|---|
| 설치 | `npx -y @agentmemory/agentmemory@latest` |
| 플러그인/스킬/MCP? | 전부. 4층 구조 (서버 → MCP → 훅 → 스킬) |
| API 키 필요? | **불필요.** `EMBEDDING_PROVIDER=local`이면 의미검색도 무료 |
| 왜 유명? | Karpathy 설계 유래 + 보편적 페인포인트 + 20개 에이전트 + 로컬 우선 |
| 로컬 에이전트 | REST로 연동하거나 구현 교본으로 활용 |
| 수익화 | Apache-2.0으로 가능. **한국 온프렘 + SLA**가 최우선 |
| React / PHP | **React로 UI 신규 제작이 최선.** PHP는 연동 레이어로 |
