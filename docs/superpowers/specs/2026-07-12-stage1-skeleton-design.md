# Stage 1 설계: 뼈대 — 해시 라우팅 · CONFIG · 모임/멤버 CRUD

**날짜:** 2026-07-12
**범위:** 오.내.총 5단계 구현 중 Stage 1만. Stage 2~5(정산 엔진, 영수증, Firebase, PWA)는 절대 선행 구현하지 않는다.
**출처:** 사용자가 제공한 "Stage 1 구현 프롬프트"(그대로 반영) + `오내총_PRD_v1.md`

이 문서는 사용자가 이미 완결된 형태로 제시한 스펙을 그대로 정리·확정한 것이다. 별도의 탐색적 질문 없이, 프롬프트에 명시된 값과 PRD의 배경을 결합해 구현 계획(writing-plans)으로 바로 넘길 수 있는 형태로 기록한다.

## 1. 기술 제약 (전 단계 공통)

1. 단일 `index.html` — HTML/CSS/Vanilla JS. 빌드 도구·npm·프레임워크 금지.
2. 외부 리소스는 CDN만. Stage 1은 웹폰트(Pretendard)만 사용.
3. 모든 상수는 파일 상단 `CONFIG` 객체에 집약. 매직 넘버/문자열 금지.
4. 주석은 한국어. 섹션 구분 주석으로 코드 구조 명확화.
5. 모바일 퍼스트 반응형: 기준 폭 390px, 최대 콘텐츠 폭 560px 중앙 정렬.
6. 배포 대상 GitHub Pages — 상대 경로만 사용.

## 2. CONFIG 객체

```javascript
const CONFIG = {
  APP_NAME: '오늘은 내가 총무',
  APP_SHORT: '오.내.총',
  STORAGE_KEY: 'onct_events',
  PREFS_KEY: 'onct_prefs',
  MEMBER_MIN: 2,
  MEMBER_MAX: 12,
  MEMBER_COLORS: [
    '#FFB4A2', '#FFD6A5', '#FDFFB6', '#CAFFBF',
    '#9BF6FF', '#A0C4FF', '#BDB2FF', '#FFC6FF',
  ],
  MEMBER_EMOJIS: ['🦊','🐰','🐻','🐹','🐱','🐶','🐧','🐥','🦁','🐸','🐼','🐨'],
  DATE_LOCALE: 'ko-KR',
  COLOR_PRIMARY: '#FF7A45',   // 여우 오렌지 계열 포인트색
  COLOR_BG: '#FFF9F2',        // 연한 크림 배경
};
```

- 파스텔 8색 팔레트(위 hex 값)와 포인트 컬러(`#FF7A45` 코랄/여우 오렌지), 배경(`#FFF9F2` 크림)을 확정값으로 사용한다. 이후 단계에서 시각적으로 조정하더라도 CONFIG 값만 바꾸면 되도록 유지한다.

## 3. 라우팅

- `location.hash` 기반 미니 라우터.
- 라우트: `#/` (홈), `#/event/{eventId}` (모임 설정). 미정의 라우트 → `#/`로 리다이렉트.
- 각 화면은 렌더 함수(`renderHome()`, `renderEventSettings(id)` 등) 방식. Stage 2~4에서 `#/event/{id}/items`, `#/view/{shareId}`를 라우트 테이블에 추가만 하면 되는 구조.

## 4. 데이터 모델 및 저장 계층

Firestore 이관(Stage 4)을 고려해 필드명을 고정한다:

```javascript
{
  id: string,              // crypto.randomUUID()
  name: string,
  date: string,            // YYYY-MM-DD
  status: 'draft',         // Stage 1은 항상 draft
  account: { bank: '', number: '', holder: '' },
  members: [
    { id, name, emoji, color, isOwner }
  ],
  items: [],               // Stage 2용, 빈 배열
  createdAt: number,
  updatedAt: number,
}
```

- `store` 추상화: `store.getAll()`, `store.get(id)`, `store.save(event)`, `store.remove(id)`.
  UI 코드는 `store`를 통해서만 데이터에 접근 — localStorage를 직접 만지지 않는다. Stage 4에서 `store` 내부 구현만 Firestore로 교체.
- `store.save()`는 `updatedAt`을 자동 갱신.

## 5. 홈 화면 (모임 목록)

- 모임 카드: 모임명, 날짜, 멤버 이모지 나열, 상태 뱃지("작성 중"), 총액 자리 표시(₩0).
- 정렬: `date` 내림차순.
- [+ 새 모임] → 생성 후 해당 모임 설정 화면으로 이동.
- 카드 탭 → 모임 설정 화면. 카드 더보기(⋯) → 확인 다이얼로그 후 삭제.
- 빈 상태: 이모지 + "첫 모임을 만들어보세요" 안내.

## 6. 모임 설정 화면 (정보 + 멤버)

- 모임 정보: 모임명(필수), 날짜(기본값 오늘), 계좌 정보 3필드(선택).
- 멤버 관리: 추가(이름 + 이모지 선택 + 색상 자동 순환 배정), 목록(칩 표시 + 총무 뱃지), 총무 지정(라디오, 기본 첫 멤버), 삭제(총무 삭제 시 첫 멤버로 자동 이관), 인원 2~12 제한.
- 자동 저장: 입력 변경 시 debounce 500ms + "저장됨 ✓" 인디케이터.
- 유효성: 모임명 공백 불가, 멤버 이름 공백·중복 불가.
- 하단 [결제 입력으로 →] 버튼은 자리만 배치, 탭 시 "Stage 2에서 열립니다" 토스트 + 비활성 스타일.

## 7. 디자인 방향

- 컨셉: 귀엽고 산뜻한 가계부. 파스텔 배경, 카드형 UI, 둥근 모서리 12~16px, 부드러운 그림자.
- 본문 폰트 Pretendard(CDN), 금액은 고정폭 느낌.
- 포인트 컬러 1색(코랄/여우 오렌지 계열), 배경 연한 크림톤. 다크모드 범위 외.
- 토스트, 확인 다이얼로그, 뱃지는 공용 함수로 구현 — 이후 단계에서 재사용.

## 8. 범위 제외

결제 항목 입력/정산 계산/표(Stage 2), 영수증/사진/html2canvas(Stage 3), Firebase/로그인/공유 링크(Stage 4), PWA manifest(Stage 5) — 이번 단계에서 구현하지 않는다.

## 9. 완료 기준 (수동 검증)

1. 모임 "여우회 강화도 나들이" 생성, 멤버 5인(인우·찬우·정표·정우·은찬, 총무=정우) 등록 → 홈 카드 확인.
2. 새로고침 후 데이터 유지(localStorage).
3. 멤버 이름 중복 입력 차단 메시지.
4. 총무 멤버 삭제 → 총무 자동 이관.
5. `#/nope` 접근 시 홈으로 리다이렉트.
6. 375px(iPhone SE)~데스크톱 레이아웃 깨짐 없음.

## 10. 테스트 전략 메모

프로젝트가 빌드 도구·npm 없이 단일 `index.html`로 제한되므로, 자동화된 유닛 테스트 프레임워크(Jest 등)는 도입하지 않는다. 대신:
- `store` 계층과 멤버/총무 이관 로직 등 순수 함수는 브라우저 콘솔에서 실행 가능한 자체 점검 함수로 검증(assert 기반, 콘솔 출력)하거나 코드 리뷰로 검증한다.
- 위 §9의 완료 기준은 실제 브라우저(다양한 뷰포트)에서 수동으로 검증하고 결과를 보고한다.

## Stage 2로 넘기는 인터페이스

- `store.getAll() / get() / save() / remove()` — Stage 2에서 `items` 필드 조작에 그대로 재사용.
- 라우터 테이블에 라우트 추가 방식 — `#/event/{id}/items` 추가 예정.
- 공용 UI 함수: `showToast()`, `confirmDialog()`, `renderBadge()` 등 — 이름은 구현 단계에서 확정, 이후 단계에서 재사용.
