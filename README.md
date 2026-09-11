# E2E_AGENT

프로젝트에 상관없이 **동일한 E2E 검증 파이프라인**으로 Playwright 테스트를 계획·생성·수리·실행하기 위한 표준 가이드입니다.

목표: SDD/스펙 기반 구현 속도에 맞춰, E2E도 **명세 → 코드 → 실행 → 복구 → 사람 리뷰**로 병목을 줄인다.

---

## 파이프라인 개요

```
[1] Spec / AC / Plan
        ↓
[2] Planner  → 시나리오 명세 (Markdown / YAML)
        ↓  (사람 리뷰)
[3] Generator → Playwright 테스트 코드
        ↓
[4] Run       → playwright test (+ trace)
        ↓
[5] Healer    → 실패 분류 후 수리 또는 리포트
        ↓  (사람 리뷰)
[6] CI Gate   → label / nightly / PR smoke
```

핵심 원칙:

- AI는 **코드 작성 보조**이지, assertion을 마음대로 약화하거나 앱 버그를 테스트로 덮지 않는다.
- 모든 프로젝트는 **같은 단계·같은 산출물 이름·같은 실패 분류**를 쓴다.
- CI에 들어가기 전 **사람 리뷰 1회**는 필수다.
- 브라우저는 **Chromium(Chrome)만**. Firefox/WebKit은 쓰지 않는다.
- 권한 검증은 브라우저를 늘리는 게 아니라 **실계정 세션(`storageState`)을 역할별로** 나눈다.

### SDD와의 관계 (시점)

시점을 한 덩어리로 보지 않는다.

| 단계 | 언제 | 하는 일 |
|------|------|---------|
| [1] Spec 묶음 | 스펙 작성·확정 | PRD + SPEC + (스펙 안의 성공/실패/예외) |
| [2] Planner | 스펙 완료 직후 (코드 전·중 OK) | 화면으로 증명할 P0만 실행형 시나리오로 |
| [3][4][5] | 해당 플로우 UI가 동작할 때 | Playwright 코드 생성·실행·테스트 수리 |
| [6] CI | 스위트가 안정된 뒤 | smoke / label / nightly |

정리:

- **PRD + SPEC + 테스트케이스(성공/실패/예외 원문)** = SDD 한 묶음.
- Playwright 코드 생성·실행은 그 묶음 다음, **구현이 그 여정을 브라우저에서 돌릴 수 있을 때**.

### 각 단계가 무엇인지

#### [1] Spec / AC / Plan — 요구사항 정본

무엇을 만들지·무엇이 맞는지 정한 문서 묶음이다. 브라우저는 돌리지 않는다.

입력 예:

- PRD / 메일 확정 요구 / 작업명세서
- Superpowers plan / 구현 플랜
- 기능 스펙 보일러플레이트 (`처리` · `성공` · `실패` · `예외`)
- 티켓 AC / 수동 QA 체크리스트

스펙 보일러플레이트에 이미 `성공` / `실패` / `예외`를 나눠 둔 경우, 그것은 **인수 조건(AC)에 해당하는 규칙**이다. 예:

- 성공: 탭을 옮기거나 새로고침해도 같은 조건이 유지된다
- 실패: 시작일이 종료일보다 늦으면 조회하지 않고 입력 오류를 표시한다
- 예외: 탭마다 조회 기준이 다르면 변환한 실제 적용 범위를 화면에 표시한다

이 문장들은 **테스트케이스 원문**이다. 아직 Playwright 시나리오 파일이 아니다.

**바로 Generator에 코드를 시키지 않는다.** 먼저 [2]로 내린다.

#### [2] Planner → 시나리오 명세 — 실행 단위로 쪼개기

스펙의 성공·실패·예외를 **통째로 E2E에 넣지 않는다.**  
그중 **화면으로 증명할 P0**만 “누가 / 어디서 / 무엇을 하면 / 뭐가 보여야 한다” 형태로 고른다.

| 스펙에 있는 것 | [2]에서의 취급 |
|----------------|----------------|
| 화면 동선·권한·버튼/메뉴 유무 | E2E 시나리오 후보 (P0만) |
| 산식·귀속·집계 규칙 | 대부분 **제외** → unit / API |
| 데이터 예외·부분 집계 문구 | API·유닛 우선, E2E는 대표 1건만 |

산출물: `e2e/specs/*.md` 또는 `*.yaml`  
사람 리뷰: 스펙 문장을 왜곡하지 않았는지, P0만 있는지, ASSUMPTION이 남아 있으면 스펙부터 보완.

#### [3] Generator → Playwright 코드 — 시나리오를 `.spec.ts`로

[2] 명세 + seed/helpers를 입력으로, **라이브 Chromium**에서 locator를 검증하며 테스트 코드를 만든다.  
해당 화면이 이미 구현돼 동작해야 한다.

#### [4] Run → `playwright test` — Chromium에서 실행

작성된 테스트를 돌리고, 실패 시 error · screenshot · **trace**를 남긴다.  
계정은 매 테스트마다 비밀번호를 치는 게 아니라, setup이 만들어 둔 **세션 JSON(`storageState`)** 을 로드한다.

#### [5] Healer → 테스트 수리 또는 리포트 (앱 코드 수정 아님)

Healer가 손대는 기본 대상은 **앱 기능 코드가 아니라 E2E 테스트 코드**(`.spec.ts` / helper)다.  
“바로 고쳐 머지”가 아니라 **패치 제안 → 가드레일 안 수정 시도 → 사람 리뷰**다.

| 분류 | 의미 | Healer 행동 |
|------|------|-------------|
| `UI_CHANGE` | UI 변경으로 locator 깨짐 | 테스트/Helper 수정 |
| `TEST_BUG` | 대기·스코프·데이터 준비 오류 | 테스트 수정 |
| `APP_BUG` | 앱 실제 결함 | **수정하지 않음**, 이슈 리포트 |
| `ENV_ISSUE` | 서버·네트워크·시드 문제 | **수정하지 않음**, 환경 리포트 |

자동 커밋·자동 머지 금지. assertion 약화 금지. 최대 3회 루프.

#### [6] CI Gate — 언제 돌릴지

전 PR 풀스위트를 기본으로 두지 않는다. smoke / label / nightly로 나눈다. Chromium만.

### 한 줄 연결

```
[1] 스펙의 처리·성공·실패·예외     ← SDD 보일러플레이트 (이미 있는 경우가 많음)
        ↓ 화면 P0만 실행형으로
[2] e2e/specs 시나리오
        ↓ 구현 후
[3][4][5][6] Playwright 코드 · 실행 · 테스트 수리 · CI
```

---

## 0. 프로젝트 공통 전제

각 앱 레포에 아래를 맞춘다.

| 항목 | 표준 |
|------|------|
| 프레임워크 | Playwright (TypeScript) |
| 브라우저 | **Chromium만** (`npx playwright install chromium`) |
| Agents | Playwright Test Agents: Planner / Generator / Healer |
| 시나리오 | `e2e/specs/` (또는 `specs/`) |
| 테스트 코드 | `e2e/tests/` (또는 `tests/`) |
| Seed | `e2e/tests/seed.spec.ts` — 로그인·fixture·환경 부트스트랩 |
| 권한 세션 | `.auth/{member,leader,director}.json` — setup이 실계정으로 생성 |
| Helper | `e2e/helpers/` — 디자인시스템/공통 UI 추상화 |
| 가이드 | `e2e/AGENTS.md` 또는 Cursor/Claude agent 정의 |

초기화 예시:

```bash
npm init playwright@latest
npx playwright install chromium
npx playwright init-agents --loop=vscode   # 또는 claude / codex
```

### 권한·세션 로드 (실계정)

팀원 / 팀장 / 본부장처럼 역할이 나뉘면:

1. env에 실계정 정보를 둔다 (`E2E_MEMBER_USER` 등). git에 비밀번호를 넣지 않는다.
2. **setup 한 번**만 그 계정으로 로그인해 `.auth/*.json`에 쿠키·localStorage를 저장한다.
3. 테스트 project가 해당 JSON을 `storageState`로 읽어 **이미 그 권한으로 로그인된 Chromium Context**에서 시작한다.

매 테스트마다 로그인 폼을 다시 치지 않는다. 서버 입장에서는 그 계정의 진짜 세션을 재사용하는 것이다.

상세 절차·파이프라인 매핑은 아래 **「권한별 실계정 테스트」** 절을 따른다.

---

## 권한별 실계정 테스트 (파이프라인 표준)

권한이 N개면 **그 권한에 맞는 실계정 N개**로 접속해 검증한다.  
예: 팀원 · 팀장 · 본부장 → 아이디 3개. mock 유저·프론트 숨김만으로 대체하지 않는다.

목표는 “버튼이 안 보이는지”만이 아니라, **그 계정으로 들어갔을 때 서버·메뉴·데이터 범위가 스펙과 같은지**다.

### 전체 흐름

```
[1] Spec에 역할×허용/거부 규칙을 적는다
        ↓
[2] Planner가 role 필드가 있는 시나리오로 쪼갠다
        ↓
    env에 역할별 실계정 준비 (CI secret / 로컬 .env)
        ↓
    auth.setup: 계정마다 로그인 → .auth/{role}.json
        ↓
[3] Generator: project(role) + storageState 기준으로 코드 생성
        ↓
[4] Run: Chromium에서 역할 project별로 동일/차별 expect 실행
        ↓
[5] Healer: 권한 실패가 APP_BUG면 테스트로 덮지 않음
        ↓
[6] CI: setup → member/leader/director project 순(또는 병렬)
```

### [1] Spec — 권한 규칙을 테스트케이스 원문에 포함

스펙(또는 공통 권한 절)에 역할별 기대를 명시한다. 예:

| 역할 | 허용 | 거부 |
|------|------|------|
| 팀원 (member) | 본인 범위 조회 | 승인·전역 지표·타 팀 데이터 |
| 팀장 (leader) | 팀 범위 조회·승인 | 본부 전역·타 팀 |
| 본부장 (director) | 상위 메뉴·넓은 조회 | (스펙에 정의된 입력 메뉴만 제한 등) |

성공/실패/예외 문장에 **누가**가 빠져 있으면 Planner가 ASSUMPTION을 남기게 하고, 스펙을 먼저 고친다.

### [2] Planner — 시나리오마다 `role` 필수

권한 E2E는 화면 하나가 아니라 **역할×여정**이다.

```yaml
id: metrics-menu-member-denied
title: 팀원은 지표 전용 메뉴에 진입할 수 없다
priority: P0
role: member
route: /tsv-metrics
steps:
  - action: 직접 URL 진입 시도
expects:
  - access: denied   # 메뉴 없음 또는 가드/리다이렉트
---
id: metrics-menu-director-ok
title: 본부장은 지표 메뉴에 진입할 수 있다
priority: P0
role: director
route: /tsv-metrics
steps:
  - action: 메뉴 또는 URL로 진입
expects:
  - page: visible
  - heading_includes: 취급고
```

규칙:

- 같은 화면이라도 역할마다 **별도 시나리오**(또는 명확한 role 매트릭스)로 둔다.
- P0만: 메뉴 진입, 핵심 버튼, 데이터 범위 대표 1건. 권한 매트릭스 전체를 E2E로 도배하지 않는다.
- UI 숨김만 expect하지 말고, 가능하면 **직접 URL 진입**도 한 시나리오에 넣는다.

### 계정 준비 (env) — 파이프라인 공통 전제

역할 수 = 계정 수. staging/전용 E2E 계정을 쓴다. prod 계정 금지.

```bash
# .env.e2e (gitignored) 또는 CI secrets
E2E_BASE_URL=https://staging.example.com

E2E_MEMBER_USER=e2e.member
E2E_MEMBER_PASSWORD=***

E2E_LEADER_USER=e2e.leader
E2E_LEADER_PASSWORD=***

E2E_DIRECTOR_USER=e2e.director
E2E_DIRECTOR_PASSWORD=***
```

| 역할 키 | 파일 | 의미 (프로젝트마다 이름만 맞춰도 됨) |
|---------|------|--------------------------------------|
| `member` | `.auth/member.json` | 팀원 |
| `leader` | `.auth/leader.json` | 팀장 |
| `director` | `.auth/director.json` | 본부장 |

`.auth/` 는 `.gitignore`. CI는 잡 시작 시 setup이 새로 만든다.

### auth.setup — 아이디로 접속해 세션 덤프 ([4] 직전 필수)

스위트(또는 CI job) 시작 시 **역할마다 1회** 로그인한다.

```
Chromium
  → 빈 Context
  → E2E_MEMBER_USER/PASSWORD 로 /login
  → 로그인 성공 확인 (홈·메뉴 로드)
  → storageState 저장 → .auth/member.json
  → leader / director 동일 반복
```

주의:

- 로그인 실패·OTP·캡차는 setup 실패로 끊는다. 본 테스트를 빈 세션으로 돌리지 않는다.
- 토큰 TTL이 짧으면 job마다 setup을 다시 돈다.
- 계정 비밀번호는 테스트 코드·trace 업로드 설명에 넣지 않는다.

### playwright projects — 역할 = 세션 슬롯

브라우저를 3개 쓰는 게 아니라 **Chromium project 3개**다.

```ts
// 개념 예시
projects: [
  { name: "setup", testMatch: /auth\.setup\.ts/ },
  {
    name: "member",
    dependencies: ["setup"],
    use: { storageState: ".auth/member.json" },
  },
  {
    name: "leader",
    dependencies: ["setup"],
    use: { storageState: ".auth/leader.json" },
  },
  {
    name: "director",
    dependencies: ["setup"],
    use: { storageState: ".auth/director.json" },
  },
]
```

Run 시:

```
npx playwright test --project=member
npx playwright test --project=leader
npx playwright test --project=director
# 또는 전체: setup 후 세 project
```

같은 `page.goto("/approvals")`라도 project가 다르면 **다른 아이디의 세션**으로 들어가고, expect만 역할에 맞게 갈라진다.

### [3] Generator — 역할 시나리오를 코드로

지시 요지:

- 시나리오의 `role` → 해당 project에서만 실행되게 작성 (`test.describe` + project grep, 또는 파일 분리).
- 테스트 본문에서 로그인 UI·계정 문자열을 다시 치지 말 것.
- member 거부 / director 허용처럼 **대칭 시나리오**가 있으면 둘 다 생성.

### [4] Run — 실제로 “그 아이디로” 검증하는 순간

세션 로드 후 동작:

1. Context가 `.auth/{role}.json`의 쿠키·localStorage 주입
2. 앱이 그 사용자를 인식
3. 메뉴·API·데이터 범위가 그 권한으로 응답
4. expect로 스펙과 비교

권한 P0 체크리스트 예:

- [ ] 팀원: 금지 메뉴 없음 + 직접 URL 차단/리다이렉트
- [ ] 팀장: 팀 범위 목록 보임, 타 팀·전역은 안 보임(또는 스펙대로)
- [ ] 본부장: 허용 메뉴 진입·조회 성공
- [ ] (필요 시) 입력 전용 계정은 지표 조회 필드가 안 내려오는지

### [5] Healer — 권한 실패 분류

| 증상 | 분류 | 행동 |
|------|------|------|
| 버튼 텍스트·locator만 바뀜 | `UI_CHANGE` | 테스트 수정 |
| 잘못된 project/storageState 경로 | `TEST_BUG` | 테스트/설정 수정 |
| 팀원인데 전사 데이터 보임 등 스펙 위반 | `APP_BUG` | **테스트 약화 금지**, 앱 이슈 리포트 |
| 계정 잠김·로그인 서버 다운 | `ENV_ISSUE` | setup/환경 리포트 |

권한 버그를 “expect를 느슨하게” 고쳐서 초록으로 만들면 파이프라인 위반이다.

### [6] CI — 역할 매트릭스

```
CI job
  1) secrets → env
  2) playwright install chromium
  3) auth.setup (계정 3 로그인 → .auth/*.json)
  4) smoke: 역할별 P0만
  5) nightly: 권한 시나리오 전체 (여전히 Chromium만)
```

artifact에 `.auth/*.json`을 올리지 않는다. trace만 실패 시 업로드.

### 하지 말 것 (권한)

- 한 계정으로 로그인 후 localStorage만 조작해 역할 흉내
- 프론트에서 메뉴 숨김만 보고 권한 통과 처리 (URL·API 우회 미검증)
- 권한 조합 전수 E2E화 (매트릭스는 API/유닛, E2E는 P0 여정)
- 실계정 비밀번호를 스펙·시나리오 YAML·커밋에 기재

---

## 1. Spec 입력 (SDD와 연결) — 상세는 위 [1]

입력·보일러플레이트·성공/실패/예외의 의미는 **파이프라인 개요 → [1]** 을 따른다.  
Generator로 바로 가지 말고 Planner([2])로 내린다.
권한 규칙은 **「권한별 실계정 테스트」** 절과 함께 스펙에 적어 둔다.
---

## 2. Planner → 시나리오 명세

### AI 지시 예시

```text
역할: Playwright Planner
입력: @docs/spec/<feature>.md , @e2e/tests/seed.spec.ts
요청: 스펙의 성공/실패/예외 중 화면 P0만 시나리오로 작성해.
출력: e2e/specs/<feature>.md (또는 .yaml)
제약:
- happy path + 중요 실패/권한 케이스만
- 산식·집계 세부는 E2E에 넣지 말 것
- 각 step에 action / expect / data / route / role(팀원|팀장|본부장) 명시
- UI 구현 디테일 추측 금지. 불확실하면 ASSUMPTION 표시
```

### 시나리오 스키마 (프로젝트 공통)

Markdown이든 YAML이든 **필드는 동일**하게 유지한다.

```yaml
id: growth-request-create-happy
title: 성장요청 생성 성공
priority: P0
role: member   # member | leader | director
preconditions:
  - 해당 역할 storageState 로드됨
  - seed 데이터가 준비됨
route: /growth/requests/new
steps:
  - action: 필수 필드 입력
    data: { account: "ACME", amount: 1000000 }
  - action: 저장 클릭
expects:
  - toast: success
  - url_includes: /growth/requests/
assumptions: []
```

사람 리뷰 체크:

- [ ] P0만 먼저 있는지
- [ ] 기대 결과가 비즈니스 동작인지 (DOM 구현 디테일이 아닌지)
- [ ] 스펙의 성공/실패/예외 문장과 추적이 되는지
- [ ] ASSUMPTION이 남아 있으면 스펙 보완 후 진행

---

## 3. Generator → Playwright 코드

### AI 지시 예시

```text
역할: Playwright Generator
입력: @e2e/specs/<feature>.md , @e2e/tests/seed.spec.ts , @e2e/helpers/
요청: 시나리오를 Playwright 테스트로 변환해. Chromium만 사용.
제약:
- Helper가 있으면 Helper만 사용
- locator 우선순위: getByRole > getByLabel > getByTestId
- CSS/XPath page.locator() 금지
- 라이브 페이지에서 locator/assertion 검증
- 역할은 storageState project에 맡기고 테스트 안에서 로그인 UI를 다시 돌리지 말 것
- 파일은 e2e/tests/<feature>.spec.ts
```

### Helper 규칙 (공통)

저수준 Playwright API를 테스트에 흩뿌리지 않는다.

```ts
// prefer
await form.fillFields({ 이름: "ACME" });
await form.submit("저장");
await toast.expectSuccess();

// avoid in specs
await page.locator(".btn-primary").click();
```

프로젝트마다 Helper 구현은 달라도, **메서드 네이밍 컨벤션은 맞춘다** (`fillFields`, `submit`, `expectSuccess` 등).

---

## 4. Run

로컬:

```bash
npx playwright test
npx playwright test e2e/tests/<feature>.spec.ts --trace on
```

실패 시 수집물 (Healer 입력으로 필수):

- error message
- screenshot / trace
- 가능하면 aria snapshot
- 실패한 테스트 소스

---

## 5. Healer → 수리 또는 리포트

**앱 코드를 고쳐 주는 단계가 아니다.** 깨진 E2E 테스트(또는 Helper)를 고치거나, 앱/환경 문제로 리포트한다.  
상세 분류·가드레일은 **파이프라인 개요 → [5]** 와 동일하다.

### 실패 분류 (모든 프로젝트 동일)

| 코드 | 의미 | AI 행동 |
|------|------|---------|
| `UI_CHANGE` | 의도적 UI 변경으로 locator 깨짐 | locator/Helper 수정 |
| `TEST_BUG` | 대기/스코프/데이터 준비 오류 | 테스트 수정 |
| `APP_BUG` | 앱 실제 결함 | **수정하지 않음**, 이슈로 리포트 |
| `ENV_ISSUE` | 서버/네트워크/시드 문제 | **수정하지 않음**, 환경 리포트 |

### AI 지시 예시

```text
역할: Playwright Healer
입력: 실패한 테스트 이름, error, trace/screenshot
요청: 실패 원인을 4분류 중 하나로 분류한 뒤 처리해.
가드레일:
- 최대 3회 수정 루프
- expect/assertion 약화 금지 (toBeTruthy로 바꾸기, timeout만 늘리기 등 금지)
- APP_BUG / ENV_ISSUE면 skip 남발하지 말고 원인과 재현 절차를 리포트
- 앱 소스 수정 금지. 테스트·Helper만 대상
- 패치는 draft로 남기고 사람 승인 전 머지 금지
```

---

## 6. CI Gate (공통 정책)

한 번에 전 PR 풀스위트를 돌리지 않는다. **Chromium만** 설치·실행한다.

| 트리거 | 범위 | 목적 |
|--------|------|------|
| PR + `e2e` label | 관련 smoke / 변경 feature | 필요할 때만 |
| PR → main (필수 경로) | P0 smoke만 | 회귀 최소 차단 |
| nightly schedule | full suite (Chromium) | 커버리지 |
| release label | full suite (Chromium) | 배포 직전 |

권장 설정:

- shard 병렬화
- flake retry는 **1회 이하** (retry로 숨기지 말 것)
- 실패 시 HTML report + trace artifact 업로드
- 메신저/PR 코멘트로 pass/fail 요약
- setup에서 역할별 `storageState` 생성 후 project 매트릭스 실행
---

## 7. 새 프로젝트 온보딩 체크리스트

1. Playwright + Chromium + `init-agents` 설치
2. 역할별 실계정 env + `auth.setup` → `.auth/*.json` (`storageState`)
3. `seed.spec.ts` / helpers 최소 세트 (form / dialog / toast / nav / table)
4. `e2e/AGENTS.md`에 selector·Helper·Healer 규칙 복사
5. 스펙의 성공/실패/예외에서 **P0 시나리오 3~5개만** Planner → 리뷰 → Generator
6. CI는 Chromium + label 기반부터 연결
7. 이후 기능은 스펙 머지 시 시나리오 명세를 같은 PR에 포함

---

## 8. 하지 말 것

- 스펙 없이 “이 페이지 E2E 전부 짜줘”
- assertion을 통과만을 위해 약화
- 공유 DB의 mutable 데이터에 의존
- Healer가 자동 커밋/자동 머지
- E2E로 유닛/계약 테스트로 충분한 로직 검증

---

## 9. 레포 역할

이 레포(`E2E_AGENT`)는 **공통 파이프라인·규칙의 소스 오브 트루스**다.

- 앱 코드는 각 서비스 레포에 둔다.
- 파이프라인/프롬프트/스키마가 바뀌면 여기 README(및 이후 템플릿)를 먼저 갱신한다.
- 각 프로젝트는 이 문서를 기준으로 `e2e/` 구조를 맞춘다.

---

## 참고

- [Playwright Test Agents](https://playwright.dev/docs/test-agents)
- Planner → Generator → Healer 순차 사용이 기본 루프다.
