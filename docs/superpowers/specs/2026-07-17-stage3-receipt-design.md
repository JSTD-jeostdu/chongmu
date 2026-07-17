# Stage 3 설계: 감성 영수증 — 디자인 · 사진 첨부 · 이미지 저장/공유

**날짜:** 2026-07-17
**범위:** 오.내.총 5단계 구현 중 Stage 3만. Firebase(Stage 4), PWA(Stage 5), 영수증 추가 테마(v2)는 구현하지 않는다.
**출처:** 사용자가 제공한 "Stage 3 구현 프롬프트"(그대로 반영) + `오내총_PRD_v1.md` + Stage 1~2에서 확정된 실제 `index.html` 인터페이스 + 브레인스토밍에서 확정한 5가지 결정

## 0. 브레인스토밍에서 확정한 결정

1. **영수증 도트 폰트**: Galmuri11 (CDN: `https://cdn.jsdelivr.net/npm/galmuri/dist/galmuri.css`, `font-family: 'Galmuri11'`)
2. **멤버 전환 UI**: 상단 탭만 (스와이프 제스처 구현하지 않음)
3. **사진 관리 시트 진입점**: 자동(용량 초과 시) + 수동(영수증 화면 상단 아이콘 버튼) 둘 다
4. **전체 저장 버튼 위치**: 영수증 화면 상단, 멤버 탭 옆
5. **사진 첨부/필터 선택 UI**: 영수증 DOM과 분리된 별도 모달(영수증 자체는 html2canvas 캡처 대상이라 UI 전용 요소를 최소화)

## 1. Stage 1~2에서 물려받는 실제 인터페이스 (변경 금지, 재사용만)

- `CONFIG`, `store.getAll()/get(id)/save(event)/remove(id)`, `escapeHtml`/`escapeAttr`
- `calcSettlement(event)` → `{ matrix, itemTotals, memberTotals, grandTotal, transfers, errors }` (순수 함수, Stage 2에서 완성)
- `showToast(message)`, `confirmDialog(message)` (Promise\<boolean\>, `.dialog-overlay`/`.dialog-box`/`.dialog-message`/`.dialog-actions` CSS), `renderBadge(text, variant)`
- `registerRoute(pattern, renderFn)`, `parseHash()`, `matchRoute(hash)`(하드코딩 정규식 매칭), `router()`
- `renderSettlementResult(params)` 안의 `.btn-receipt` 버튼(현재 `showToast('Stage 3에서 열립니다')`만 연결) — Stage 3에서 실제 라우팅으로 교체
- `onDeleteEvent(id)` (`index.html` 내 함수, `store.remove(id)` 직후 지점) — 사진 정리 훅 추가 지점
- 모임 데이터 모델: `event.members[].{id,name,emoji,color,isOwner}`, `event.items[].{id,name,emoji,amount,participantIds,fixedAmounts,memo,order}`, `event.account.{bank,number,holder}`

Stage 3는 **새 Firestore 필드나 `event.*` 스키마 변경을 추가하지 않는다** — 사진은 전적으로 `localStorage`에만 존재하고 `event` 객체와는 `eventId:memberId` 키로만 연결된다 (CLAUDE.md의 하이브리드 저장 모델 그대로).

## 2. CONFIG 확장

```javascript
PHOTOS_KEY: 'onct_photos',
RECEIPT_WIDTH: 360,
RECEIPT_SCALE: 2,
PHOTO_MAX_EDGE: 800,
PHOTO_QUALITY: 0.7,
RECEIPT_FONT_URL: 'https://cdn.jsdelivr.net/npm/galmuri/dist/galmuri.css',
RECEIPT_MENTS: [
  '오늘도 함께해줘서 고마워요',
  '교환·환불은 다음 모임에서만 가능합니다',
  '본 영수증은 추억 보증서를 겸합니다',
  '재방문 시 웃음 10% 추가 적립',
  '분실 시 총무가 슬퍼합니다',
  '이 영수증은 세금 공제와 무관합니다',
],
PHOTO_NOTICE_KEY: 'onct_photo_notice_seen',
```

`<head>`에 두 CDN 리소스 추가: 기존 Pretendard `<link>` 옆에 Galmuri CSS `<link>`, `</body>` 직전에 html2canvas `<script src="https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js"></script>`.

## 3. 데이터 계층 — `photoStore`

`store` 정의 바로 아래, 같은 스타일로 추가:

```javascript
const photoStore = {
  _readAll() {
    const raw = localStorage.getItem(CONFIG.PHOTOS_KEY);
    return raw ? JSON.parse(raw) : {};
  },
  _writeAll(map) {
    localStorage.setItem(CONFIG.PHOTOS_KEY, JSON.stringify(map));
  },
  _key(eventId, memberId) { return `${eventId}:${memberId}`; },
  get(eventId, memberId) {
    return this._readAll()[this._key(eventId, memberId)] || null;
  },
  set(eventId, memberId, dataUrl) {
    const map = this._readAll();
    map[this._key(eventId, memberId)] = dataUrl;
    this._writeAll(map); // QuotaExceededError는 호출부(첨부 모달)에서 try/catch
  },
  remove(eventId, memberId) {
    const map = this._readAll();
    delete map[this._key(eventId, memberId)];
    this._writeAll(map);
  },
  removeByEvent(eventId) {
    const map = this._readAll();
    Object.keys(map).forEach(k => { if (k.startsWith(`${eventId}:`)) delete map[k]; });
    this._writeAll(map);
  },
  listByEvent(eventId) {
    const map = this._readAll();
    return Object.keys(map).filter(k => k.startsWith(`${eventId}:`)).map(k => ({ key: k, memberId: k.split(':')[1], dataUrl: map[k] }));
  },
  usage() {
    // 바이트 근사치 = JSON 문자열 길이 (UTF-16 코드유닛 수, base64라 ASCII라서 실질적으로 바이트 수와 근접)
    const raw = localStorage.getItem(CONFIG.PHOTOS_KEY) || '';
    return Math.round(raw.length / 1024); // KB
  },
};
```

`set()`이 `QuotaExceededError`를 던지면 호출부(첨부 모달의 확정 버튼 핸들러)가 잡아서: 저장 취소 + `showToast('사진 용량이 가득 찼어요. 지난 모임 사진을 정리해주세요')` + `openPhotoManagerSheet()` 자동 오픈.

## 4. 사진 압축·필터 파이프라인

순수 함수로 유틸 섹션에 추가 (DOM 부수효과 없음, `<canvas>`만 사용):

```javascript
function resizeImageToCanvas(file, maxEdge) → Promise<HTMLCanvasElement>
// FileReader → Image → 긴 변 기준 maxEdge로 축소한 canvas 반환 (원본 비율 유지)

function applyPhotoFilter(sourceCanvas, filterName) → HTMLCanvasElement
// filterName: 'original' | 'grayscale' | 'dither'
// 'original': sourceCanvas를 그대로 복제
// 'grayscale': getImageData → luminance(0.299R+0.587G+0.114B) 변환 + 대비 소폭 상승(감열지 감성) → putImageData
// 'dither': grayscale 결과에 4×4 Bayer 행렬 ordered dithering 적용 → 흑/백 2톤
//   (구현 중 복잡도가 과할 경우 grayscale + 랜덤 노이즈 오버레이로 대체하고 최종 보고서에 명시 — 프롬프트가 허용한 폴백)

function canvasToCompressedDataURL(canvas, quality) → string
// canvas.toDataURL('image/jpeg', quality)
```

**원본 이중 저장 금지**: `resizeImageToCanvas` 결과(리사이즈된 원본, 압축 전)만 모듈 스코프 변수(`pendingPhotoCanvas`)에 유지하며 필터 3종 미리보기를 이 변수에서 매번 재생성한다. `localStorage`에는 사용자가 필터를 확정한 **결과물 1개**만 `photoStore.set`으로 저장한다.

## 5. 영수증 화면 & 라우팅

`matchRoute`에 기존 두 패턴과 같은 방식으로 추가:

```javascript
const receiptMatch = hash.match(/^\/event\/([^/]+)\/receipt\/([^/]+)$/);
if (receiptMatch && routes['/event/:id/receipt/:memberId']) {
  return { render: routes['/event/:id/receipt/:memberId'], params: { id: receiptMatch[1], memberId: receiptMatch[2] } };
}
```

`registerRoute('/event/:id/receipt/:memberId', renderReceiptView)`.

`renderReceiptView(params)`:
- `.topbar`: 뒤로가기(`#/event/{id}/result`로 이동) + "영수증" 타이틀
- **멤버 탭 바** (`.receipt-member-tabs`, 가로 스크롤 `overflow-x: auto`): `ev.members.map(m => 탭 버튼)`, 활성 탭은 `params.memberId`와 일치하는 멤버. 탭 클릭 시 `location.hash = '#/event/{id}/receipt/{m.id}'`로 교체 이동(재렌더는 `hashchange` 리스너가 자동 처리, 별도 재호출 불필요)
- 탭 바 옆에 아이콘 버튼 2개: `[🖼 사진 관리]`(→ `openPhotoManagerSheet(ev.id)`), `[⬇ 전체 저장]`(→ `saveAllReceipts(ev)`)
- 탭 바 아래 `.receipt-stage`(어두운 배경 컨테이너, §7) 안에 영수증 DOM 1장
- 영수증 아래 액션 버튼 3개: `[이미지 저장]` / `[이미지 공유]` / (계좌 있으면) `[계좌 복사]`

`params.memberId`가 `ev.members`에 없으면(멤버 삭제 후 남은 딥링크 등) `ev.members[0]`으로 폴백하거나, 멤버가 0명이면 결과 화면으로 리다이렉트.

## 6. 영수증 DOM 구조 & 데이터 바인딩

`renderReceiptDom(ev, member, result)` — `renderReceiptView`에서 호출되는 순수 렌더 함수(문자열 반환), 캡처 대상 루트가 `<div class="receipt" id="receipt-capture">`.

**데이터 산출**:
- 참여 항목 = `ev.items.filter(item => result.matrix[item.id] && result.matrix[item.id][member.id] !== undefined)`, `order`로 정렬
- 각 항목의 표시 금액 = `result.matrix[item.id][member.id]`
- `*` 표시 여부 = `item.fixedAmounts && item.fixedAmounts[member.id] !== undefined`
- 합계 = `result.transfers[member.id] || 0` (총무가 아닌 경우), 총무면 스탬프로 대체
- 영수증 번호 = `event.id.slice(0,8).toUpperCase()` + `-` + `(ev.members.indexOf(member)+1)`을 2자리 zero-pad
- 발행시각 = 렌더 시점 `new Date()` (실물 영수증처럼 "지금 발급"하는 개념 — 항목 수정 이력과 무관하므로 캐싱하지 않는다)

**구조 (8섹션, 프롬프트 순서 그대로)**:
1. `.receipt-shop`: `오늘은 내가 총무`(큰 글씨) / `{ev.name}점` / `대표: {총무 이름}`
2. `.receipt-meta`: 날짜(`ev.date`), 발행시각, `NO. {영수증번호}`
3. `.receipt-photo-slot`: `photoStore.get(ev.id, member.id)`가 있으면 `<img>` 폭 100%, 없으면 점선 박스 `📷 하이라이트 사진 추가` — 클릭 시 `openPhotoAttachModal(ev.id, member.id)`. **이 슬롯 자체는 `data-no-capture` 대상이 아님(사진은 캡처되어야 함)** — 다만 미첨부 상태의 "탭해서 추가" 안내 텍스트는 캡처에는 남아도 무방(빈 점선 박스 자체가 실물 영수증의 재미 요소).
4. `.receipt-items`: 각 항목 `{emoji} {name} ....... ₩{amount}{fixed? '*':''}`(점선 리더는 `flex` + `overflow:hidden` + 반복 `.` 텍스트 또는 `border-bottom: 1px dotted`), 고정 항목 있으면 하단에 `* 직접 조정된 금액` 각주
5. `.receipt-total`: 굵은 점선 구분선 → 총무가 아니면 `합계 ₩{transfers[member.id]}`(큰 글씨), **총무면** `.receipt-stamp`(기울어진 빨간 테두리 텍스트 "결제 완료", `transform: rotate(-8deg)`, `border: 3px solid #D64545`, `color: #D64545`)
6. `.receipt-account` (계좌 정보 있고 총무 아닐 때만): `{은행} {계좌번호} ({예금주})` + `[복사]` 버튼(`data-no-capture` — 캡처에서 제외)
7. `.receipt-barcode`: `<svg>` 세로줄 ~40개, 폭은 `hashString(ev.id + member.id)` 기반 결정론적 의사난수로 생성(새로고침해도 동일 모양) + 아래 작은 글씨로 `ev.id.slice(0,12)`
8. `.receipt-ment`: `RECEIPT_MENTS[hashString(ev.id+member.id) % RECEIPT_MENTS.length]`(같은 사람은 항상 같은 멘트) + `⁂ 오.내.총 ⁂`

`hashString(str)`은 간단한 문자열 해시(예: 32비트 FNV-1a 변형) — 유틸 섹션에 추가, 바코드 폭 시드와 멘트 선택 인덱스 양쪽에 재사용.

## 7. 영수증 CSS 디자인

- `.receipt-stage`(영수증을 감싸는 배경): 어두운 톤(`background: #2B2420`), `padding`, 영수증에 `box-shadow: 0 12px 40px rgba(0,0,0,0.5)`로 실물처럼 떠 보이게.
- `.receipt`: `width: {CONFIG.RECEIPT_WIDTH}px`, `background: #fdfdf8`, `color: #2A231C`, `font-family: 'Galmuri11', monospace`, `margin: 0 auto`.
- 절취선(상/하단): `.receipt-tear-top`/`.receipt-tear-bottom` — `height: 10px`에 `background: linear-gradient(135deg, #2B2420 25%, transparent 25%) 0 0/10px 10px repeat-x` 같은 지그재그 패턴(정확한 각도/크기는 구현 중 시각 확인하며 조정, 스펙은 "지그재그 절취선 느낌"까지만 고정).
- 금액: `toLocaleString('ko-KR')`, `font-variant-numeric: tabular-nums`로 고정폭 정렬.
- 좁은 화면(`RECEIPT_WIDTH=360`)이 375px 뷰포트보다는 작으므로 별도 반응형 축소는 불필요 — `.receipt-stage`가 `overflow-x: auto`만 가지면 충분(초소형 기기 대비).

## 8. 사진 첨부 모달

`openPhotoAttachModal(eventId, memberId)` — `confirmDialog`와 같은 "동적 오버레이 생성" 패턴이지만 폭이 더 필요하므로 새 CSS 클래스 `.sheet-overlay`(배경은 `.dialog-overlay`와 동일) + `.sheet-box`(`width: min(400px, 92vw)`, `max-height: 85vh`, `overflow-y: auto`) 사용.

흐름:
1. 최초 호출 시(전역 `localStorage.getItem(CONFIG.PHOTO_NOTICE_KEY)` 없음) → 안내 `confirmDialog` 스타일 알림 먼저 표시("사진은 이 기기에만 저장돼요...") → 확인 시 `localStorage.setItem(CONFIG.PHOTO_NOTICE_KEY, '1')` 후 계속 진행
2. `<input type="file" accept="image/*">` 트리거 (`capture` 속성은 넣지 않음 — 넣으면 카메라 앱이 바로 열려 갤러리 선택지가 가려짐. 하이라이트 사진은 대부분 모임 중 미리 찍어둔 사진을 나중에 갤러리에서 고르는 흐름이라 OS 기본 선택지(갤러리/카메라/파일)를 그대로 노출하는 쪽이 실사용에 맞음)
3. 파일 선택 시 `resizeImageToCanvas` → `pendingPhotoCanvas`에 저장 → 3개 필터 미리보기 썸네일(각각 `applyPhotoFilter(pendingPhotoCanvas, filterName)` 결과를 작은 `<canvas>`로 렌더) 표시, 기본 선택은 `original`
4. 필터 썸네일 클릭으로 선택 전환(라디오 형태 UI)
5. `[확정]` 버튼 → 선택된 필터의 canvas를 `canvasToCompressedDataURL` → `photoStore.set(eventId, memberId, dataUrl)`(try/catch로 QuotaExceededError 처리) → 모달 닫고 `renderReceiptView` 재호출
6. `[취소]` / 오버레이 바깥 클릭 → `pendingPhotoCanvas = null` 후 모달만 닫기 (기존 사진 유지)
7. 이미 사진이 있는 상태에서 슬롯 클릭 시에도 동일 모달이 뜨되, 상단에 `[현재 사진 삭제]` 버튼 추가(클릭 시 `confirmDialog` 재확인 → `photoStore.remove` → 재렌더)

## 9. 사진 관리 시트

`openPhotoManagerSheet(eventId)` — `.sheet-overlay`/`.sheet-box` 재사용. 내용:
- 상단: `photoStore.usage()`를 이용한 "전체 사진 용량: 약 {N}KB" 표시(브라우저 localStorage 한도는 기기마다 다르므로 절대치 경고선은 두지 않고 참고용 숫자만 표시)
- `photoStore.listByEvent(eventId)` 결과를 썸네일 그리드로 나열(다른 모임 사진은 이 시트에서 다루지 않음 — 현재 모임 범위로 한정, YAGNI). 각 썸네일에 멤버 이름 + `[삭제]` 버튼(`confirmDialog` 재확인 후 `photoStore.remove`)
- 하단: `[이 모임 사진 전체 삭제]` 버튼(`confirmDialog` 재확인 후 `photoStore.removeByEvent`)
- 진입점: 영수증 화면 상단 아이콘 버튼(수동) + `photoStore.set`에서 `QuotaExceededError` 캐치 시 자동 호출(자동 호출 시에는 안내 토스트가 먼저 뜬 뒤 시트가 열림)

## 10. 캡처 · 공유 · 저장 파이프라인

```javascript
async function captureReceiptToBlob(receiptEl) {
  await document.fonts.ready;
  const canvas = await html2canvas(receiptEl, {
    scale: CONFIG.RECEIPT_SCALE,
    backgroundColor: null,
    ignoreElements: (el) => el.dataset.noCapture !== undefined,
  });
  return new Promise(resolve => canvas.toBlob(resolve, 'image/png'));
}
```

- **`[이미지 저장]`**: `captureReceiptToBlob` → `URL.createObjectURL(blob)` → 임시 `<a download="오내총_{ev.name}_{member.name}.png">` 클릭 트리거 → `revokeObjectURL`.
- **`[이미지 공유]`**: `captureReceiptToBlob` → `File` 객체 생성 → `navigator.canShare && navigator.canShare({files:[file]})`이면 `navigator.share({files:[file]})`. 미지원이면 저장과 동일한 다운로드 폴백 실행 + `showToast('갤러리에서 공유해주세요')`.
- **`[전체 저장]`**: `ev.members`를 순회하며 각 멤버 탭으로 전환(`location.hash` 변경 없이 내부적으로 `renderReceiptView`를 해당 `memberId`로 직접 호출해 DOM만 교체 — 매 반복 실제 라우팅을 타면 `hashchange` 이벤트 중복 처리가 꼬일 수 있으므로 내부 렌더 함수를 직접 재사용) → 각 캡처 사이 `await new Promise(r => setTimeout(r, 150))`로 레이아웃/폰트 안정화 대기 → 순차 다운로드. 진행 상태는 `showToast('영수증 저장 중... ({i}/{n})')`로 매 스텝 갱신(토스트가 겹치면 기존 토스트를 즉시 교체하는 기존 `showToast` 동작을 그대로 사용).
- 계좌 `[복사]`: Stage 2 결과 화면과 동일 패턴(`navigator.clipboard.writeText` + `showToast`) 재사용, 버튼 자체는 `data-no-capture`.

## 11. 삭제 연동

`onDeleteEvent(id)` 안, `store.remove(id)` 다음 줄에 `photoStore.removeByEvent(id);` 한 줄 추가.

## 12. 범위 제외

Firebase/로그인/공유 링크(Stage 4), PWA manifest(Stage 5), 영수증 추가 테마/스킨(v2), 스와이프 제스처(브레인스토밍에서 탭만으로 확정), 사진 관리 시트에서 타 모임 사진 일괄 조회(현재 모임 범위로 한정).

## 13. 완료 기준 검증 방식

Chrome 브라우저 확장이 이번 세션(Stage 2 작업분)에서 계속 연결 실패했다 — Stage 3 착수 시 재시도하되, 계속 실패하면 Stage 2와 동일하게 jsdom + 실제 DOM 이벤트 구동으로 검증한다. 단, **html2canvas의 실제 래스터라이즈 결과(폰트 렌더링 품질, 픽셀 단위 캡처 정확도)는 jsdom에서 신뢰할 수 없다** — 이 부분(완료 기준 3번의 "720px 폭, 폰트 깨짐 없음")은 jsdom에서는 "캡처 함수가 에러 없이 Blob을 반환하고 크기가 0이 아님"까지만 확인하고, 실제 시각적 품질 확인은 브라우저 가능 시로 명확히 구분해 보고한다.

프롬프트가 제시한 8개 완료 기준을 순서대로 실행하며, 각 항목의 검증 방법(jsdom vs 실브라우저)을 최종 보고에 명시한다.

## Stage 4로 넘기는 인터페이스 메모

- `photoStore`는 Stage 4에서도 그대로 유지된다(사진은 Firebase Storage로 옮기지 않는 것이 CLAUDE.md의 확정 설계) — Stage 4 작업 시 이 함수들의 시그니처를 변경하지 않는다.
- 영수증 화면(`renderReceiptView`)은 Stage 4의 `/#/view/{shareId}` 공개 읽기 전용 화면에서는 재사용하지 않는다(CLAUDE.md: 공유 뷰는 사진을 표시하지 않고 텍스트 정산 데이터만 노출) — 별도 화면으로 분리될 예정, Stage 3 범위 아님.
