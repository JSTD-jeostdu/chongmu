# Stage 2 설계: 결제 항목 입력 · 정산 엔진 · 결과 표

**날짜:** 2026-07-17
**범위:** 오.내.총 5단계 구현 중 Stage 2만. Stage 3~5(영수증, Firebase, PWA)는 절대 선행 구현하지 않는다.
**출처:** 사용자가 제공한 "Stage 2 구현 프롬프트"(그대로 반영) + `오내총_PRD_v1.md` + Stage 1에서 확정된 실제 `index.html` 인터페이스

이 문서는 사용자가 이미 완결된 형태로 제시한 스펙을 실제 Stage 1 코드(현재 `index.html`, 743줄)에 맞춰 정리·확정한 것이다. 함수명·CSS 클래스는 실제 코드를 기준으로 한다.

## 0. Stage 1에서 물려받는 실제 인터페이스 (변경 금지, 재사용만)

- `CONFIG` 객체 (APP_NAME, STORAGE_KEY, MEMBER_MIN/MAX, MEMBER_COLORS, MEMBER_EMOJIS, DATE_LOCALE, COLOR_PRIMARY, COLOR_BG)
- `store.getAll() / get(id) / save(event) / remove(id)` — `save()`가 `updatedAt` 자동 갱신
- `escapeHtml(str)`, `escapeAttr(str)` — XSS 방지 유틸
- `createEvent`, `nextMemberColor`, `addMember`, `removeMember`, `setOwner` — 도메인 함수 섹션
- `runSelfTests()` — 콘솔 자체 테스트 (Stage 1 전용, Stage 2는 별도 `runRegressionTests()` 추가)
- `showToast(message)`, `confirmDialog(message)` (Promise<boolean>), `renderBadge(text, variant)` — 공용 UI
- `registerRoute(pattern, renderFn)`, `parseHash()`, `matchRoute(hash)`, `router()` — 라우터. `matchRoute`는 라우트별 정규식을 하드코딩하는 방식(범용 패턴 매처 아님)
- `renderEventSettings(params)` 내부의 `flushSave(ev)` / `scheduleSave(ev)` debounce 패턴, `saveDebounceTimer` 전역 변수
- CSS 클래스: `.card-section`, `.btn`/`.btn-primary`/`.btn-ghost`/`.btn-danger`/`.btn-disabled`, `.field-label`/`.input`/`.field-error`/`.field-hint`, `.badge-*`, `.topbar`/`.btn-back`/`.save-indicator`
- `#btn-goto-items` 버튼(현재 `renderEventSettings` 안에 있고 토스트만 띄움) — Stage 2에서 실제 라우트 이동으로 교체

## 1. CONFIG 확장

기존 `CONFIG` 객체에 아래 필드 추가 (프롬프트 그대로):

```javascript
ROUNDING: 'floor1',            // 'floor1' | 'ceil10' | 'ceil100'
ITEM_EMOJIS: ['🍽️','☕','🍦','📷','🚗','🎮','🎫','🛍️','🍺','🧾'],
CURRENCY_LOCALE: 'ko-KR',
DEV_MODE: false,
```

Stage 1에서 확정한 "주요 색상 3개만 CONFIG" 원칙은 유지 — 결과 화면의 파란 하이라이트 등은 리터럴 CSS 색상으로 추가하고 CONFIG에 넣지 않는다.

## 2. 데이터 모델 — `event.items[]`

```javascript
{
  id: string,                    // crypto.randomUUID()
  name: string,                  // 항목명 (필수)
  emoji: string,                 // 선택 ('' 허용) — ITEM_EMOJIS 선택 또는 자유 입력
  amount: number,                 // 원 단위 정수 (0 허용)
  participantIds: [memberId],     // 최소 1인 (저장 시 강제)
  fixedAmounts: {},               // { memberId: 정수 } — 지정된 경우만 키 존재
  memo: string,
  order: number,                  // 정렬 순서 (배열 인덱스와 별개로 유지, 순서 변경 시 갱신)
}
```

`fixedAmounts`가 비어있으면 전원 균등분배. 값이 일부만 있으면 나머지 참여자가 잔여를 균등분배.

## 3. 정산 계산 엔진 — `calcSettlement(event)`

Stage 1의 도메인 함수들(`createEvent` 등)과 같은 섹션에 순수 함수로 추가. `store`나 DOM에 의존하지 않는다.

```javascript
calcSettlement(event) → {
  matrix: { [itemId]: { [memberId]: number } },
  itemTotals: { [itemId]: number },     // = item.amount
  memberTotals: { [memberId]: number },
  grandTotal: number,
  transfers: { [memberId]: number },    // 총무 제외, 피청구인만
  errors: [ { itemId, code, message } ],
}
```

### 분배 규칙

1. `fixedAmounts`에 지정된 참여자 → 지정 금액 그대로 `matrix[item.id][memberId]`에 기록.
2. 나머지 참여자 → `잔여 = amount − Σ고정금액`을 `CONFIG.ROUNDING`에 따라 분배.

### 반올림 정책

**`floor1`(기본):**
- 몫 = `Math.floor(잔여 / 나머지인원수)`
- 끝전 = `잔여 − 몫 × 나머지인원수`
- 끝전 귀속 대상 결정: 해당 항목의 `participantIds`에 **총무(`event.members`에서 `isOwner===true`인 멤버)가 포함되어 있으면 총무에게 가산**. 총무가 미참여면 **`event.members` 배열 순서상 `participantIds`에 포함된 가장 앞선 멤버**에게 가산 (별도 `order` 필드 불필요 — 멤버 배열 자체의 등록 순서를 사용).

**`ceil10` / `ceil100`:**
- 총무 외 참여자 몫 = 균등액을 10원/100원 단위로 올림(`Math.ceil(x/10)*10` 등)
- 총무 몫 = `amount − Σ(다른 참여자 몫) − Σ고정금액`. 음수가 되면 해당 항목만 `floor1` 규칙으로 폴백하고 `console.warn`으로 항목 id를 남긴다.
- 총무가 해당 항목에 미참여면 `floor1` 규칙 적용(올림 정책은 총무 존재를 전제로 하므로).

### 무결성 검증 (매 항목마다)

- `participantIds.length === 0` → `NO_PARTICIPANT`
- `Σ(fixedAmounts 값) > amount` → `FIXED_EXCEEDS`
- `fixedAmounts`의 키 중 `participantIds`에 없는 것 → `FIXED_NOT_PARTICIPANT`
- 위 오류가 없는 항목에 한해 `Σ(matrix[itemId] 값) === item.amount` assert. 실패 시 `INTEGRITY_MISMATCH` 에러로 push (이론상 발생하면 안 되지만 방어적으로 남긴다).
- `errors`가 비어있지 않아도 저장은 항상 허용. 결과 화면에서 "정산 완료" 전환만 막는다 (`status`를 `'settled'`로 바꾸는 버튼 비활성 + 사유 텍스트 표시).

## 4. 라우터 확장

`matchRoute`에 기존 `/event/:id` 패턴과 같은 방식으로 두 정규식을 추가:

```javascript
const itemsMatch = hash.match(/^\/event\/([^/]+)\/items$/);
if (itemsMatch && routes['/event/:id/items']) { return { render: routes['/event/:id/items'], params: { id: itemsMatch[1] } }; }
const resultMatch = hash.match(/^\/event\/([^/]+)\/result$/);
if (resultMatch && routes['/event/:id/result']) { return { render: routes['/event/:id/result'], params: { id: resultMatch[1] } }; }
```

범용 패턴 매처로 리팩터링하지 않는다 (라우트 2개 추가에 불과, YAGNI).

## 5. 결제 입력 화면 — `#/event/{id}/items`

- `renderEventSettings`와 동일한 화면 골격: `.topbar`(뒤로가기 + `모임명` 타이틀 + `.save-indicator`) + `.screen` 안에 `.card-section`들.
- **항목 리스트**: 각 항목을 카드로 렌더링 — 이모지+이름, `₩amount` (천단위 콤마), 참여자 이모지 나열(미참여자는 `opacity: 0.35`로 반투명 처리), `fixedAmounts` 키가 있으면 "직접 조정" 뱃지.
- **추가/수정 폼 (인라인, 카드 하단에 배치)**:
  - 항목명 입력(필수)
  - 이모지 피커: `CONFIG.ITEM_EMOJIS` 그리드 + 자유 입력용 텍스트 필드 1개(입력 시 그 값이 우선, `escapeHtml`로 렌더링 — 자유 입력이라 Stage 1 멤버 이모지와 달리 XSS 이스케이프 필수)
  - 금액 입력: `<input type="text" inputmode="numeric">`, `input` 이벤트에서 숫자만 추출해 천단위 콤마 포맷, 커서 위치는 끝으로 고정(포맷팅 후 커서 튐 방지 — 커서를 항상 끝에 두는 단순한 전략 채택, 복잡한 커서 위치 보존 로직은 만들지 않음)
  - 참여자 체크: 멤버 칩 토글(기본 전원 선택) + `[전체 선택]`/`[전체 해제]` 버튼
  - 고급 옵션(접이식, `<details>` 또는 토글 버튼): 참여자별 금액 직접 입력 필드. 일부만 입력 시 나머지는 실시간 미리보기("잔여 ₩X를 N명이 나눔 → 1인 ₩Y")로 안내. 고정 합이 금액을 초과하면 즉시 `.field-error`에 표시하고 저장 차단(프롬프트 §3 "저장 시점 이전에 즉시 차단"과 동일 정책을 폼에서도 선반영)
  - 메모 입력(선택, `<input>` 한 줄)
- 항목 삭제: `confirmDialog` 재사용. 순서 변경: 위/아래 버튼만 구현 (드래그는 만들지 않음 — 프롬프트가 "선택 구현"으로 명시).
- 화면 하단 고정 바(`.item-summary-bar`, 새 CSS): 멤버 이모지 + 실시간 인당 누적 금액(`calcSettlement` 호출 결과의 `memberTotals`) + `[정산 결과 보기 →]` 버튼.
- 자동저장: 기존 `flushSave`/`scheduleSave`와 동일한 debounce(500ms) + "저장됨 ✓" 패턴을 이 화면 전용으로 재사용(같은 `saveDebounceTimer` 전역 변수 재사용 — 화면이 다르면 어차피 이전 타이머는 의미 없으므로 문제 없음).

## 6. 정산 결과 화면 — `#/event/{id}/result`

- 표: 행 = 항목(이모지+이름), 열 = 멤버(이모지+이름), 셀 = 분배액 또는 `—`(미참여). 마지막 열 = 항목 합계(`itemTotals`).
- 하단 총계 행: `memberTotals` 하이라이트(리터럴 CSS: 파란 배경 + 흰 글씨), 우하단 = `grandTotal`.
- 표 컨테이너: `overflow-x: auto`, 첫 열(`<th>`/`<td>` 항목명) `position: sticky; left: 0;`.
- 표 아래 송금 요약 카드: 총무 제외 멤버별 `이모지 이름 → ₩transfers[memberId]`. 총무 행은 "결제 완료 🧾" 텍스트로 대체.
- 계좌 정보(`event.account`)가 하나라도 있으면 표시 + `[복사]` 버튼(`navigator.clipboard.writeText`, 성공 시 `showToast`).
- `[정산 완료]` 버튼: `errors.length > 0`이면 비활성 + 사유(`.field-error`류 텍스트로 첫 에러 메시지 표시). 활성 상태에서 클릭 시 `event.status = 'settled'`, `store.save`, 홈 카드 뱃지 갱신.
- **`'draft'` 자동 복귀**: 정산 완료 후 항목 화면(`#/event/{id}/items`)에서 항목을 추가/수정/삭제/삭제하면 저장 시점에 `event.status === 'settled'`인 경우 `'draft'`로 되돌린다 (items 화면의 저장 경로에 이 로직을 둔다).
- 멤버별 영수증 버튼: 자리만 배치, 클릭 시 `showToast('Stage 3에서 열립니다')`.

## 7. 홈 화면 반영

`renderEventCard`(기존 함수)에서 `calcSettlement(ev).grandTotal`을 계산해 현재의 `₩0` 자리 표시를 실제 값으로 교체. `items`가 비어있으면 자연히 0이 나오므로 별도 분기 불필요.

## 8. 회귀 테스트 — `runRegressionTests()`

Stage 1의 `runSelfTests()`와 나란히 배치하되 별도 함수. `CONFIG.DEV_MODE === true`일 때 `DOMContentLoaded` 시점에 자동 실행하고 `console.table`로 출력(Stage 1 패턴과 동일).

프롬프트가 지정한 시드 A/B + 단위 테스트 5종을 그대로 구현한다:

1. 시드 A(항목 10개) → 인당 ₩39,820 / 총계 ₩199,100
2. 시드 B(항목 6개) → 인당 ₩54,800 / 총계 ₩274,000
3. 끝전 단위 테스트: 10,000원 ÷ 3인 → 3,334/3,333/3,333
4. 참여자 제외 단위 테스트: 시드 B '방탈출' 75,000원에서 은찬 제외 → 4인 각 18,750
5. 고정 금액 단위 테스트: '더벤티' 15,700원에서 정표 고정 3,640 → 나머지 4인 각 3,015
6. 에러 검출 단위 테스트: 고정 합 초과 → `FIXED_EXCEEDS`
7. 모든 케이스에서 `Σ분배액 === amount` assert

**추가(프롬프트에 없지만 보강)**: `ceil10`/`ceil100` 분기 각 최소 1개 단위 테스트 — 정산 엔진에서 코드로는 존재하지만 테스트되지 않는 분기를 남기지 않기 위함. 시드 데이터와는 무관한 독립 테스트 케이스로 추가.

## 9. 범위 제외

영수증 UI/사진/html2canvas(Stage 3), Firebase/공유 링크(Stage 4), PWA(Stage 5) — 구현하지 않는다.

## 10. 완료 기준 (수동 검증)

1. `runRegressionTests()` 전체 통과 (시드 A/B 정확한 값 + 단위 테스트 5종 + ceil10/ceil100 보강 테스트)
2. UI에서 시드 A 데이터 수동 입력 → 결과 표가 스프레드시트와 동일 구조로 렌더링
3. 항목에서 참여자 1인 제외 → 표에 `—` + 인당 금액 재계산 확인
4. 금액 직접 조정 → 잔여 자동 균등 + 고정 합 초과 시 에러 확인
5. `[정산 완료]` → 홈 뱃지 변경 → 항목 수정 → `작성 중` 자동 복귀 확인
6. 375px 화면에서 표 가로 스크롤 + 첫 열 sticky 동작 확인

## Stage 3로 넘기는 인터페이스

- `calcSettlement(event)` 반환 객체 전체 — 영수증은 `matrix`/`memberTotals`/`transfers`를 그대로 소비.
- 결과 화면의 멤버별 영수증 버튼 자리(현재 토스트만 연결) — Stage 3에서 실제 렌더링으로 교체.
