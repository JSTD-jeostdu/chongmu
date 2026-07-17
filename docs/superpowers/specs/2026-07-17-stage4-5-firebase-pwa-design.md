# Stage 4+5 설계: Firebase 동기화 · 공유 뷰 · 마감

**날짜:** 2026-07-17
**범위:** 오.내.총 5단계 구현 중 Stage 4(Firebase Auth·Firestore 동기화·공유 뷰·Security Rules)와 Stage 5(PWA manifest·반응형 마감·안내 문구)를 통합 진행. v2 백로그는 범위 외.
**출처:** 사용자가 제공한 "Stage 4+5 통합 프롬프트(압축판)" + `오내총_PRD_v1.md` §4.6/§5/§6/§7 + Stage 1~3에서 확정된 실제 `index.html` 인터페이스 + 브레인스토밍에서 확정한 결정

## 0. 브레인스토밍에서 확정한 결정

1. **Google 로그인 참고 구현**: 없음 — 표준 Firebase Auth(`signInWithPopup`)로 새로 구현
2. **store 동기화 아키텍처**: 메모리 캐시 + write-through 방식 (기존 `store.*` 호출부 20여 곳 무변경)
3. **공유 뷰 재사용 전략**: 정산 표·송금 카드 마크업을 `renderSettlementTable`/`renderTransferCard`로 분리해 정산 결과 화면과 공유 뷰가 공유
4. **아이콘**: 별도 SVG 파일(`icons/icon.svg`) — data URL 아님, `apple-touch-icon`/파비콘/매니페스트 공통 참조
5. **로딩 화면**: 두지 않음 — 로컬 캐시로 즉시 렌더 후 로그인 상태면 백그라운드에서 Firestore 재조회·필요 시만 재렌더
6. **Firebase 미설정 시**: `CONFIG.FIREBASE.apiKey`가 빈 문자열이면 Firebase 초기화 자체를 건너뛰고 로그인 버튼을 비활성 표시 — 나머지 기능은 100% 로컬 모드로 정상 동작
7. **shareId**: 로그인 여부와 무관하게 이벤트 **생성 시점에 항상** 부여

## 1. Stage 1~3에서 물려받는 실제 인터페이스 (변경 금지, 재사용만)

- `CONFIG`, `escapeHtml`/`escapeAttr`, `showToast(message)`, `confirmDialog(message)`
- `calcSettlement(event)` → `{ matrix, itemTotals, memberTotals, grandTotal, transfers, errors }` (순수 함수, 절대 수정하지 않음)
- `registerRoute(pattern, renderFn)`, `parseHash()`, `matchRoute(hash)`(하드코딩 정규식), `router()`
- `photoStore`(전 메서드 시그니처 그대로 — Stage 4에서도 사진은 `localStorage`에만 존재, Firebase Storage로 옮기지 않음)
- `renderReceiptView`/`renderReceiptDom` 등 영수증 관련 함수 — 공유 뷰에서 재사용하지 않음(공유 뷰는 사진·영수증 없이 텍스트 정산 데이터만 노출)
- 모임 데이터 모델: `event.{id,name,date,status,account:{bank,number,holder},members[],items[],createdAt,updatedAt}`

이 스테이지는 `event.*` 스키마에 `ownerUid`(선택, 클라우드 동기화된 이벤트만), `shareId`(모든 이벤트에 항상) 두 필드만 추가한다. 그 외 필드/구조는 변경하지 않는다.

## 2. CONFIG 확장

```javascript
FIREBASE: {
  apiKey: '',
  authDomain: '',
  projectId: '',
  storageBucket: '',
  messagingSenderId: '',
  appId: '',
},
MIGRATION_FLAG_PREFIX: 'onct_migrated_', // + uid
APP_THEME_COLOR: undefined, // 사용하지 않음 — COLOR_PRIMARY를 manifest에서 직접 재사용
```

`CONFIG.FIREBASE`의 6개 필드는 전부 빈 문자열 placeholder로 커밋한다. 주석으로 "Firebase 콘솔 > 프로젝트 설정 > 웹 앱에서 발급받은 값을 채워주세요" 명시.

`<head>`에 CDN 3개 추가(Firebase compat, 버전 10.13.2 고정):
```html
<script src="https://www.gstatic.com/firebasejs/10.13.2/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.13.2/firebase-auth-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.13.2/firebase-firestore-compat.js"></script>
```
기존 html2canvas `<script>` 옆, 메인 인라인 스크립트보다 앞에 위치.

`icons/icon.svg`(신규 파일), `manifest.json`(신규 파일) 참조용 `<link>` 4개도 `<head>`에 추가(§9).

## 3. Firebase 초기화 & 인증

```javascript
let firebaseApp = null;
let auth = null;
let firestore = null;
let currentUser = null;      // firebase.User | null
let firebaseReady = false;   // CONFIG.FIREBASE.apiKey가 채워져 있고 initializeApp 성공했는지

function initFirebase() {
  if (!CONFIG.FIREBASE.apiKey) return false; // 플레이스홀더 상태 — 조용히 스킵
  try {
    firebaseApp = firebase.initializeApp(CONFIG.FIREBASE);
    auth = firebase.auth();
    firestore = firebase.firestore();
    firestore.enablePersistence().catch(() => {}); // 다중 탭 등 실패는 무시
    firebaseReady = true;
    return true;
  } catch (err) {
    console.warn('Firebase 초기화 실패, 로컬 모드로 동작합니다:', err);
    return false;
  }
}
```

- 로그인: `auth.signInWithPopup(new firebase.auth.GoogleAuthProvider())`
- 로그아웃: `auth.signOut()`
- `auth.onAuthStateChanged(user => { currentUser = user; ... })` — 부트스트랩(§5)과 홈 화면 로그인 버튼 재렌더를 트리거하는 단일 지점
- 로그인 버튼은 `renderHome()`의 topbar 우측에: `firebaseReady`가 false면 비활성 버튼 + title="Firebase 설정이 필요해요", true이고 `currentUser`가 없으면 "Google로 로그인", 있으면 프로필 사진(있으면) + 이름 첫 글자 + "로그아웃"

## 4. store 재구현 — 메모리 캐시 + write-through

```javascript
let eventsCache = [];

function loadLocalCache() {
  eventsCache = store._readAllLocal();
}

const store = {
  _readAllLocal() {
    const raw = localStorage.getItem(CONFIG.STORAGE_KEY);
    return raw ? JSON.parse(raw) : [];
  },
  _writeAllLocal() {
    localStorage.setItem(CONFIG.STORAGE_KEY, JSON.stringify(eventsCache));
  },
  getAll() { return eventsCache.slice(); },
  get(id) { return eventsCache.find(e => e.id === id) || null; },
  save(event) {
    event.updatedAt = Date.now();
    if (!event.shareId) event.shareId = generateShareId();
    const idx = eventsCache.findIndex(e => e.id === event.id);
    if (idx === -1) eventsCache.push(event); else eventsCache[idx] = event;
    this._writeAllLocal();
    if (currentUser && firebaseReady) scheduleCloudSave(event);
    return event;
  },
  remove(id) {
    eventsCache = eventsCache.filter(e => e.id !== id);
    this._writeAllLocal();
    if (currentUser && firebaseReady) {
      firestore.collection('events').doc(id).delete().catch(err => console.warn('클라우드 삭제 실패:', err));
    }
  },
};

function generateShareId() {
  return crypto.randomUUID().replace(/-/g, '').slice(0, 20);
}

let cloudSaveTimers = {}; // eventId -> timer, 이벤트별 독립 debounce
function scheduleCloudSave(event) {
  clearTimeout(cloudSaveTimers[event.id]);
  cloudSaveTimers[event.id] = setTimeout(() => {
    const payload = { ...event, ownerUid: currentUser.uid };
    firestore.collection('events').doc(event.id).set(payload, { merge: true })
      .catch(err => console.warn('클라우드 저장 실패(로컬엔 이미 반영됨):', err));
  }, 800);
}
```

- **기존 20여 곳의 `store.get/getAll/save/remove` 호출부는 전부 무수정.**
- `save()`는 항상 동기적으로 `eventsCache`+`localStorage`를 갱신한 뒤 반환 — 화면은 항상 즉시 반응. 클라우드 반영은 그 뒤에 비동기·디바운스(800ms, CLAUDE.md 명시값)로 베스트-에포트 수행.
- 클라우드 저장 실패는 사용자에게 별도로 알리지 않는다(로컬엔 이미 반영되어 데이터 손실이 없고, `console.warn`으로만 남긴다) — 재시도 로직은 이번 스테이지 범위 밖(YAGNI, 다음 저장이나 새로고침 때 자연히 재시도됨).

## 5. 부트스트랩 & 로그인 시 마이그레이션

```javascript
function bootstrapApp() {
  loadLocalCache();     // 동기 — 항상 즉시 사용 가능
  router();             // 로컬 캐시로 즉시 첫 렌더 (로딩 화면 없음)

  if (!initFirebase()) return; // 플레이스홀더 상태면 여기서 끝 — 완전한 로컬 모드

  auth.onAuthStateChanged(async (user) => {
    const wasLoggedIn = !!currentUser;
    currentUser = user;
    if (user) {
      await syncFromCloud(user);
    } else if (wasLoggedIn) {
      loadLocalCache(); // 로그아웃: 캐시를 로컬 상태로 되돌림
    }
    router(); // 현재 화면 재렌더 (로그인 버튼 상태 갱신 포함)
  });
}

async function syncFromCloud(user) {
  const snapshot = await firestore.collection('events').where('ownerUid', '==', user.uid).get()
    .catch(err => { console.warn('클라우드 조회 실패, 로컬 데이터 유지:', err); return null; });
  if (!snapshot) return; // 오프라인 등 — 로컬 캐시 그대로 사용

  const cloudEvents = snapshot.docs.map(d => d.data());
  const cloudIds = new Set(cloudEvents.map(e => e.id));

  const migrationKey = CONFIG.MIGRATION_FLAG_PREFIX + user.uid;
  if (!localStorage.getItem(migrationKey)) {
    const localOnly = eventsCache.filter(e => !cloudIds.has(e.id));
    for (const ev of localOnly) {
      const payload = { ...ev, ownerUid: user.uid, shareId: ev.shareId || generateShareId() };
      await firestore.collection('events').doc(ev.id).set(payload).catch(err => console.warn('마이그레이션 실패:', ev.id, err));
      cloudEvents.push(payload);
    }
    localStorage.setItem(migrationKey, '1');
    if (localOnly.length > 0) showToast(`기존 모임 ${localOnly.length}개를 클라우드에 백업했어요`);
  }

  eventsCache = cloudEvents;
  store._writeAllLocal(); // 로컬 미러도 최신 클라우드 상태로 동기화
}
```

- `where('ownerUid','==',uid)` 쿼리이므로 Firestore 복합 색인이 필요 없다(단일 필드 equality).
- 마이그레이션은 **합집합 업로드만 수행** — 클라우드에 이미 있는 문서를 로컬 값으로 덮어쓰지 않는다(다른 기기에서 만든 최신 데이터를 실수로 되돌리지 않기 위함).
- `bootstrapApp()`이 기존 `window.addEventListener('DOMContentLoaded', router)`를 대체한다.

## 6. 공유 뷰 화면 (`#/view/{shareId}`)

`matchRoute`에 추가:
```javascript
const viewMatch = hash.match(/^\/view\/([^/]+)$/);
if (viewMatch && routes['/view/:shareId']) {
  return { render: routes['/view/:shareId'], params: { shareId: viewMatch[1] } };
}
```
`registerRoute('/view/:shareId', renderShareView)`.

`renderShareView(params)`은 **비동기** — `store`(캐시)를 거치지 않고 Firestore에서 직접 조회(로그인 여부 무관, `firebaseReady`가 false면 즉시 "공유 기능이 아직 설정되지 않았어요" 안내):

```javascript
async function renderShareView(params) {
  const app = document.getElementById('app');
  app.innerHTML = `<main class="screen"><p class="field-hint">불러오는 중...</p></main>`;
  if (!firebaseReady) {
    app.innerHTML = `<main class="screen"><div class="empty-state"><p>공유 기능이 아직 설정되지 않았어요</p></div></main>`;
    return;
  }
  const snapshot = await firestore.collection('events').where('shareId', '==', params.shareId).limit(1).get()
    .catch(() => null);
  if (!snapshot || snapshot.empty) {
    app.innerHTML = `<main class="screen"><div class="empty-state"><p>모임을 찾을 수 없어요</p></div></main>`;
    return;
  }
  const ev = snapshot.docs[0].data();
  const result = calcSettlement(ev);
  app.innerHTML = `
    <header class="topbar">
      <h1>${escapeHtml(ev.name)}</h1>
      <span class="badge-readonly">🔒 읽기 전용</span>
    </header>
    <main class="screen">
      ${renderSettlementTable(ev, result)}
      ${renderTransferCard(ev, result)}
    </main>
  `;
  const copyBtn = document.getElementById('btn-copy-account');
  if (copyBtn) {
    copyBtn.addEventListener('click', () => {
      const text = `${ev.account.bank} ${ev.account.number} ${ev.account.holder}`.trim();
      navigator.clipboard.writeText(text).then(() => showToast('계좌 정보를 복사했어요'));
    });
  }
}
```

### 리팩터링: `renderSettlementTable`/`renderTransferCard` 추출

`renderSettlementResult`(index.html 현재 구현, 정산 표+송금 카드 마크업 인라인) 안의 두 블록을 그대로 함수로 뽑아 순수 문자열 반환 함수로 만들고, `renderSettlementResult`와 `renderShareView` 양쪽에서 호출한다. 로직/마크업 변경 없이 위치만 이동 — 순수 리팩터링.

```javascript
function renderSettlementTable(ev, result) { /* 기존 .settlement-table-wrap 블록 그대로 */ }
function renderTransferCard(ev, result) { /* 기존 .card-section(멤버별 송금 요약) 블록 그대로, hasAccount 계좌 복사 버튼 포함 */ }
```

`renderSettlementResult`는 정산 완료 버튼·영수증 버튼 등 편집 전용 UI는 그대로 자기 자신에 남긴다.

### 정산 결과 화면에 공유 링크 버튼 추가

`renderSettlementResult` 안, `renderTransferCard(ev, result)` 호출 결과 바로 아래(계좌 정보 박스가 있든 없든 항상 노출)에 추가 — `renderTransferCard` 자체에는 포함하지 않는다(공유 뷰가 이 함수를 재사용하는데, 읽기 전용 방문자에게는 공유 링크 복사 버튼이 의미가 없으므로):
```html
${currentUser && ev.ownerUid ? `<button type="button" id="btn-copy-share-link" class="btn btn-ghost">🔗 공유 링크 복사</button>`
  : `<div class="field-hint">로그인하면 공유 링크를 만들 수 있어요</div>`}
```
클릭 시 `location.origin + location.pathname + '#/view/' + ev.shareId`를 클립보드에 복사 + `showToast('공유 링크를 복사했어요')`.

## 7. Security Rules

신규 파일 `firestore.rules.txt`(배포 자동화 없음 — 사용자가 Firebase 콘솔의 Firestore Rules 편집기에 직접 붙여넣는다):

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /events/{eventId} {
      allow read: if true;
      allow create: if request.auth != null && request.auth.uid == request.resource.data.ownerUid;
      allow update, delete: if request.auth != null && request.auth.uid == resource.data.ownerUid;
    }
  }
}
```

- 읽기는 완전 공개(공유 뷰가 비로그인으로 조회해야 하므로) — `shareId`는 추측 불가한 20자 랜덤값이라는 것이 유일한 접근 제어(PRD §8 명시 정책).
- 쓰기는 문서 소유자만 — `create`와 `update/delete`를 분리한 이유는 생성 시점엔 `resource.data`(기존 문서)가 없어 `request.resource.data`(새로 쓰려는 데이터)를 봐야 하기 때문.

## 8. 안내 문구

`renderEventSettings`의 계좌 정보 필드(`input-bank`/`input-number`/`input-holder`) 아래에 신규 `field-hint` 추가:
```html
<div class="field-hint">공유 링크와 영수증에 노출됩니다</div>
```
사진 첨부 1회 안내(`showPhotoNoticeOnce`, Stage 3 구현)는 이번 스테이지에서 코드 변경 없음 — 완료 기준 검증 시 문구만 재확인.

## 9. PWA 마감

### 아이콘 — `icons/icon.svg` (신규 파일)
`COLOR_PRIMARY`(#FF7A45) 배경의 둥근 사각형 위에 🧾 이모지를 중앙 배치한 512×512 viewBox SVG. 예:
```html
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512">
  <rect width="512" height="512" rx="96" fill="#FF7A45"/>
  <text x="256" y="320" font-size="280" text-anchor="middle">🧾</text>
</svg>
```
(정확한 `y` 오프셋·`font-size`는 구현 중 시각 확인하며 미세 조정 — 이모지가 사각형 중앙에 시각적으로 맞으면 됨)

### `manifest.json` (신규 파일)
```json
{
  "name": "오늘은 내가 총무",
  "short_name": "오.내.총",
  "start_url": "./index.html",
  "display": "standalone",
  "background_color": "#FFF9F2",
  "theme_color": "#FF7A45",
  "icons": [{ "src": "icons/icon.svg", "sizes": "any", "type": "image/svg+xml", "purpose": "any" }]
}
```
`name`/`short_name`/`background_color`/`theme_color` 값은 `CONFIG.APP_NAME`/`CONFIG.APP_SHORT`/`CONFIG.COLOR_BG`/`CONFIG.COLOR_PRIMARY`와 반드시 동일한 값으로 맞춘다(정적 JSON이라 JS에서 CONFIG를 직접 참조할 수는 없으므로, 값이 CONFIG와 어긋나지 않도록 구현 시 대조).

`<head>`에 추가:
```html
<link rel="manifest" href="manifest.json">
<link rel="apple-touch-icon" href="icons/icon.svg">
<link rel="icon" href="icons/icon.svg" type="image/svg+xml">
<meta name="theme-color" content="#FF7A45">
```

### 반응형 375px 실브라우저 점검
Chrome 자동화로 6개 화면(홈, 모임 설정, 결제 입력, 정산 결과, 영수증, 공유 뷰) 각각을 375px 뷰포트에서 스크린샷 확인. 실제로 잘리거나 겹치는 요소가 있으면 그 자리에서 CSS 수정(새 기능 추가 아님, 기존 CSS 미세 조정에 한정).

## 10. 범위 제외

Firebase Storage(사진 업로드), 실시간 리스너(`onSnapshot`)를 통한 즉시 반영, 공유 링크 재발급(PRD §8에 "제공"이라 언급되어 있으나 이번 압축 프롬프트에 명시되지 않아 제외 — v2 또는 후속 요청 시), 다중 로그인 계정 간 이벤트 이전, 서비스워커, 여러 아이콘 해상도(PNG 192/512 등 — SVG 단일 파일로 대체), 계좌번호/멤버 실명에 대한 추가 마스킹·암호화.

## 11. 완료 기준 검증 방식 (중요 — 이번 세션의 근본적 제약)

이번 세션에는 **실제 Firebase 프로젝트가 존재하지 않는다**(사용자가 구현 완료 후 직접 생성). 따라서:

- **jsdom + Firestore/Auth 목(mock) 객체로 검증 가능**: `store` 캐시 로직(§4), 마이그레이션 합집합 로직(§5)의 순수 로직 부분 — `firebase.auth()`/`firebase.firestore()`를 간단한 가짜 객체로 대체해 `syncFromCloud`/`scheduleCloudSave`/`store.save`의 분기·흐름이 올바른지 단위 검증한다.
- **실브라우저로 완전히 검증 가능**: 로컬 전용 모드(비로그인) 전체 동작 무회귀, `initFirebase()`가 빈 CONFIG.FIREBASE에서 안전하게 스킵되는지, PWA manifest/아이콘 로드, 375px 반응형, 계좌 안내 문구.
- **이번 세션에 검증 불가능(구조·코드 리뷰로만 확인, 라이브 테스트는 사용자 몫)**: 실제 Google 로그인 팝업 흐름, 실제 Firestore 문서 생성·읽기, 크로스 디바이스 반영, `/#/view/{shareId}` 실데이터 조회, Security Rules가 실제로 미소유자 쓰기를 막는지.
- 완료 후 보고서에 이 구분을 명확히 표시하고, 사용자가 Firebase 프로젝트 설정 후 직접 실행할 **수동 테스트 체크리스트**를 별도로 제공한다(원 프롬프트가 요청한 "완료 후 체크리스트"와 통합).

## Stage 4+5 이후(v2) 메모

- 공유 링크 재발급, Firebase Storage 사진 마이그레이션, 실시간 동기화(`onSnapshot`)는 필요성이 확인되면 v2 백로그로 별도 브레인스토밍.
