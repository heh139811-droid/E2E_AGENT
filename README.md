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

---

## 0. 프로젝트 공통 전제

각 앱 레포에 아래를 맞춘다.

| 항목 | 표준 |
|------|------|
| 프레임워크 | Playwright (TypeScript) |
| Agents | Playwright Test Agents: Planner / Generator / Healer |
| 시나리오 | `e2e/specs/` (또는 `specs/`) |
| 테스트 코드 | `e2e/tests/` (또는 `tests/`) |
| Seed | `e2e/tests/seed.spec.ts` — 로그인·fixture·환경 부트스트랩 |
| Helper | `e2e/helpers/` — 디자인시스템/공통 UI 추상화 |
| 가이드 | `e2e/AGENTS.md` 또는 Cursor/Claude agent 정의 |

초기화 예시:

```bash
npm init playwright@latest
npx playwright init-agents --loop=vscode   # 또는 claude / codex
```

---

## 1. Spec 입력 (SDD와 연결)

입력으로 쓸 수 있는 것:

- PRD / 기능 스펙
- Superpowers plan / 구현 플랜
- Acceptance Criteria / 티켓 본문
- 기존 수동 QA 체크리스트

**바로 Generator에 코드를 시키지 않는다.** 먼저 시나리오 명세로 내린다.

---

## 2. Planner → 시나리오 명세

### AI 지시 예시

```text
역할: Playwright Planner
입력: @docs/spec/<feature>.md , @e2e/tests/seed.spec.ts
요청: 핵심 사용자 플로우만 시나리오로 작성해.
출력: e2e/specs/<feature>.md (또는 .yaml)
제약:
- happy path + 중요 실패/권한 케이스만
- 각 step에 action / expect / data / route 명시
- UI 구현 디테일 추측 금지. 불확실하면 ASSUMPTION 표시
```

### 시나리오 스키마 (프로젝트 공통)

Markdown이든 YAML이든 **필드는 동일**하게 유지한다.

```yaml
id: growth-request-create-happy
title: 성장요청 생성 성공
priority: P0
preconditions:
  - 로그인된 AE 계정
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
- [ ] ASSUMPTION이 남아 있으면 스펙 보완 후 진행

---

## 3. Generator → Playwright 코드

### AI 지시 예시

```text
역할: Playwright Generator
입력: @e2e/specs/<feature>.md , @e2e/tests/seed.spec.ts , @e2e/helpers/
요청: 시나리오를 Playwright 테스트로 변환해.
제약:
- Helper가 있으면 Helper만 사용
- locator 우선순위: getByRole > getByLabel > getByTestId
- CSS/XPath page.locator() 금지
- 라이브 페이지에서 locator/assertion 검증
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
- 패치는 draft로 남기고 사람 승인 전 머지 금지
```

---

## 6. CI Gate (공통 정책)

한 번에 전 PR 풀스위트를 돌리지 않는다.

| 트리거 | 범위 | 목적 |
|--------|------|------|
| PR + `e2e` label | 관련 smoke / 변경 feature | 필요할 때만 |
| PR → main (필수 경로) | P0 smoke만 | 회귀 최소 차단 |
| nightly schedule | full suite | 커버리지 |
| release label | full + critical browsers | 배포 직전 |

권장 설정:

- shard 병렬화
- flake retry는 **1회 이하** (retry로 숨기지 말 것)
- 실패 시 HTML report + trace artifact 업로드
- 메신저/PR 코멘트로 pass/fail 요약

---

## 7. 새 프로젝트 온보딩 체크리스트

1. Playwright + `init-agents` 설치
2. `seed.spec.ts`로 인증/환경 부트스트랩
3. `e2e/helpers/` 최소 세트 (form / dialog / toast / nav / table)
4. `e2e/AGENTS.md`에 selector·Helper·Healer 규칙 복사
5. 첫 기능은 **P0 시나리오 3~5개만** Planner → 리뷰 → Generator
6. CI는 label 기반부터 연결
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
