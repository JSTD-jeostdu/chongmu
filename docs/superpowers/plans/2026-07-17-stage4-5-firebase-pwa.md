# Stage 4+5 Implementation Plan: Firebase 동기화 · 공유 뷰 · PWA 마감

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Google 로그인 + Firestore 동기화(로그인 시), 읽기 전용 공유 뷰(`#/view/{shareId}`), Firestore Security Rules 텍스트, PWA manifest/아이콘, 반응형 마감, 안내 문구를 `index.html`(및 신규 정적 파일 2개, 텍스트 파일 1개)에 추가한다.

**Architecture:** `store`를 메모리 캐시(`eventsCache`) + write-through 방식으로 재구현해 기존 20여 곳의 동기 호출부를 전혀 건드리지 않는다. 로그인 상태에서만 백그라운드로 Firestore에 비동기 반영(디바운스 800ms)하고, 로그인 시 1회 마이그레이션(로컬 전용 이벤트만 클라우드에 업로드, 기존 클라우드 데이터는 절대 덮어쓰지 않음)을 수행한다. 공유 뷰는 `store`를 거치지 않고 Firestore를 직접 쿼리하는 별도 비동기 화면이다. `CONFIG.FIREBASE.apiKey`가 비어있으면 Firebase 관련 코드 전체가 안전하게 스킵되고 앱은 완전한 로컬 모드로 동작한다(현재 커밋 상태가 정확히 이 상태 — 실제 Firebase 프로젝트는 사용자가 이후 별도로 생성).

**Tech Stack:** Vanilla JS (기존과 동일), Firebase JS SDK 10.13.2 (compat 모드, CDN), Firestore, Firebase Auth(Google).

## Global Constraints

- CDN: `https://www.gstatic.com/firebasejs/10.13.2/firebase-{app,auth,firestore}-compat.js` (compat 모드, 정확히 이 버전 문자열 사용)
- `CONFIG.FIREBASE` 6개 필드(`apiKey, authDomain, projectId, storageBucket, messagingSenderId, appId`)는 전부 빈 문자열로 커밋 — 실제 값은 사용자가 나중에 채운다.
- `CONFIG.MIGRATION_FLAG_PREFIX = 'onct_migrated_'` (+ uid로 완성되는 localStorage 키)
- 클라우드 쓰기 디바운스: 800ms (PRD §4.6 명시값)
- `shareId` 생성: `crypto.randomUUID().replace(/-/g, '').slice(0, 20)` — 20자
- 기존 `store.getAll()/get(id)/save(event)/remove(id)` 시그니처는 절대 변경하지 않는다 — 호출부 무수정이 이번 스테이지의 핵심 제약.
- `calcSettlement(event)`는 절대 수정하지 않는다.
- 사진(`photoStore`)은 이번 스테이지에서 전혀 건드리지 않는다 — Firebase Storage로 옮기지 않는다.
- 주석은 한국어. 단일 `index.html` 유지(신규 정적 파일 `manifest.json`, `icons/icon.svg`, `firestore.rules.txt`는 예외 — 빌드 산출물이 아닌 배포용 정적 리소스/문서이므로 허용).
- Security Rules는 코드로 배포하지 않는다 — `firestore.rules.txt`는 사용자가 Firebase 콘솔에 직접 붙여넣는 텍스트일 뿐이다.
- 이번 세션에는 실제 Firebase 프로젝트가 없다 — Firebase 관련 로직은 목(mock) 객체 기반 jsdom 테스트로 검증하고, 실제 로그인/크로스디바이스 동작은 코드 리뷰로만 확인한다(설계 스펙 §11).

---

## Task 1: CONFIG.FIREBASE + CDN + initFirebase 골격 + Security Rules 문서

**Files:**
- Modify: `index.html:620-621` (CDN 스크립트), `index.html:656-657` (CONFIG), `index.html:667-669` (Firebase 초기화 함수 신규 삽입)
- Create: `firestore.rules.txt`

**Interfaces:**
- Produces: `CONFIG.FIREBASE`(object), `CONFIG.MIGRATION_FLAG_PREFIX`(string), `initFirebase()`(function, returns boolean, 부수효과로 모듈 스코프 `auth`/`firestore`/`firebaseReady` 설정), `signInWithGoogle()`/`signOutUser()`(function), `generateShareId()`(function, returns 20자 string), 모듈 스코프 변수 `auth`, `firestore`, `currentUser`(초기값 `null`), `firebaseReady`(초기값 `false`)
- Consumes: 없음(이 태스크가 Stage 4의 최하위 기반)

- [ ] **Step 1: CDN 스크립트 3개 추가**

`index.html`의 다음 블록을 찾는다(620-621번째 줄):

Find:
```html
<script src="https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js"></script>
<script>
```

Replace:
```html
<script src="https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.13.2/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.13.2/firebase-auth-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.13.2/firebase-firestore-compat.js"></script>
<script>
```

- [ ] **Step 2: CONFIG에 FIREBASE·MIGRATION_FLAG_PREFIX 필드 추가**

Find:
```javascript
    PHOTO_NOTICE_KEY: 'onct_photo_notice_seen',
  };
```

Replace:
```javascript
    PHOTO_NOTICE_KEY: 'onct_photo_notice_seen',
    FIREBASE: {
      // Firebase 콘솔 > 프로젝트 설정 > 웹 앱에서 발급받은 값을 채워주세요.
      // apiKey가 비어있으면 앱 전체가 로컬 전용 모드로 동작합니다(initFirebase 참고).
      apiKey: '',
      authDomain: '',
      projectId: '',
      storageBucket: '',
      messagingSenderId: '',
      appId: '',
    },
    MIGRATION_FLAG_PREFIX: 'onct_migrated_',
  };
```

- [ ] **Step 3: Firebase 초기화 · 인증 함수 골격 추가**

Find:
```javascript
  document.head.appendChild(receiptFontLink);

  // ====== 유틸 ======
```

Replace:
```javascript
  document.head.appendChild(receiptFontLink);

  // ====== FIREBASE 초기화 & 인증 (Stage 4) ======
  let auth = null;
  let firestore = null;
  let currentUser = null;    // firebase.User | null
  let firebaseReady = false; // CONFIG.FIREBASE.apiKey가 채워져 있고 initializeApp이 성공했는지

  function initFirebase() {
    if (!CONFIG.FIREBASE.apiKey) return false; // 플레이스홀더 상태 — 조용히 로컬 전용 모드로 스킵
    try {
      firebase.initializeApp(CONFIG.FIREBASE);
      auth = firebase.auth();
      firestore = firebase.firestore();
      firestore.enablePersistence().catch(() => {}); // 다중 탭 등으로 실패해도 무시
      firebaseReady = true;
      return true;
    } catch (err) {
      console.warn('Firebase 초기화 실패, 로컬 모드로 동작합니다:', err);
      return false;
    }
  }

  function signInWithGoogle() {
    if (!firebaseReady) return;
    auth.signInWithPopup(new firebase.auth.GoogleAuthProvider())
      .catch(err => { if (err.code !== 'auth/popup-closed-by-user') showToast('로그인에 실패했어요'); });
  }

  function signOutUser() {
    if (!firebaseReady) return;
    auth.signOut();
  }

  function generateShareId() {
    return crypto.randomUUID().replace(/-/g, '').slice(0, 20);
  }

  // ====== 유틸 ======
```

- [ ] **Step 4: `firestore.rules.txt` 생성**

레포 루트(`index.html`과 같은 위치)에 새 파일 생성:

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

이 파일은 배포 스크립트로 처리되지 않는다 — 사용자가 Firebase 콘솔의 Firestore Rules 편집기에 직접 붙여넣는 참고용 텍스트다. 파일 상단에 그 취지를 설명하는 주석 한 줄을 Rules 문법 밖(파일 맨 위, `rules_version` 앞)에 추가하지 않는다 — Rules 파일 자체는 Firebase 콘솔에 그대로 복사-붙여넣기 되므로 안내 문구를 섞으면 안 된다. 대신 이 파일을 만들 때 커밋 메시지나 태스크 리포트에 용도를 설명한다.

- [ ] **Step 5: 검증**

Node로 `initFirebase()`가 `CONFIG.FIREBASE.apiKey`가 빈 값일 때 `firebase` 전역 객체를 전혀 건드리지 않고 안전하게 `false`를 반환하는지 확인한다(이 시점에는 아직 회귀 테스트에 추가하지 않는다 — 그건 Task 2에서 `store`/동기화 로직과 함께 한 번에 추가한다). 대신 정적으로 확인: `initFirebase` 함수 본문이 `if (!CONFIG.FIREBASE.apiKey) return false;`를 `firebase.initializeApp` 호출보다 먼저 실행하는지 코드를 눈으로 재확인하고, `node -e "new Function(require('fs').readFileSync('index.html','utf-8').match(/<script>([\s\S]*)<\/script>/)[1])"` 같은 방식으로 문법 오류가 없는지만 확인한다(index.html에서 메인 인라인 `<script>` 블록을 추출해 `new Function(...)`으로 파싱 — 실행은 하지 않고 구문 검증만).

- [ ] **Step 6: 커밋**

```bash
git add index.html firestore.rules.txt
git commit -m "Stage 4: Firebase CDN·CONFIG.FIREBASE·initFirebase 골격 + Security Rules 문서"
```

---

## Task 2: store 재구현(메모리 캐시 + write-through) + 부트스트랩 + 로그인 시 마이그레이션

**Files:**
- Modify: `index.html:690-721`(기존 `store` 정의 전체를 교체), `index.html:1266`(회귀 테스트에 신규 assertion 추가), `index.html:2386`(`DOMContentLoaded` 리스너를 `bootstrapApp`으로 교체)

**Interfaces:**
- Consumes: `initFirebase()`, `auth`, `firestore`, `currentUser`, `firebaseReady`, `generateShareId()`(Task 1), `showToast(message)`, `router()`(기존)
- Produces: `store.getAll()/get(id)/save(event)/remove(id)`(시그니처 무변경, 내부만 교체), `loadLocalCache()`, `bootstrapApp()`, `syncFromCloud(user)` — 이후 태스크(Task 3~5)가 `currentUser`/`firebaseReady`/`bootstrapApp`을 그대로 소비한다.

- [ ] **Step 1: `store` 전체 교체**

Find(현재 `store` 정의 전체 — `// ====== STORE ...` 주석부터 닫는 `};`까지):
```javascript
  // ====== STORE (localStorage 추상화, Stage 4에서 Firestore로 교체 예정) ======
  const store = {
    _readAll() {
      const raw = localStorage.getItem(CONFIG.STORAGE_KEY);
      return raw ? JSON.parse(raw) : [];
    },
    _writeAll(events) {
      localStorage.setItem(CONFIG.STORAGE_KEY, JSON.stringify(events));
    },
    getAll() {
      return this._readAll();
    },
    get(id) {
      return this._readAll().find(e => e.id === id) || null;
    },
    save(event) {
      const events = this._readAll();
      event.updatedAt = Date.now();
      const idx = events.findIndex(e => e.id === event.id);
      if (idx === -1) {
        events.push(event);
      } else {
        events[idx] = event;
      }
      this._writeAll(events);
      return event;
    },
    remove(id) {
      const events = this._readAll().filter(e => e.id !== id);
      this._writeAll(events);
    },
  };
```

Replace:
```javascript
  // ====== STORE (메모리 캐시 + write-through, Stage 4: 로그인 시 Firestore로도 비동기 반영) ======
  let eventsCache = [];
  let cloudSaveTimers = {}; // eventId -> timer, 이벤트별 독립 디바운스(800ms)

  const store = {
    _readAllLocal() {
      const raw = localStorage.getItem(CONFIG.STORAGE_KEY);
      return raw ? JSON.parse(raw) : [];
    },
    _writeAllLocal() {
      localStorage.setItem(CONFIG.STORAGE_KEY, JSON.stringify(eventsCache));
    },
    getAll() {
      return eventsCache.slice();
    },
    get(id) {
      return eventsCache.find(e => e.id === id) || null;
    },
    save(event) {
      event.updatedAt = Date.now();
      if (!event.shareId) event.shareId = generateShareId();
      const idx = eventsCache.findIndex(e => e.id === event.id);
      if (idx === -1) {
        eventsCache.push(event);
      } else {
        eventsCache[idx] = event;
      }
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

  function loadLocalCache() {
    eventsCache = store._readAllLocal();
  }

  function scheduleCloudSave(event) {
    clearTimeout(cloudSaveTimers[event.id]);
    cloudSaveTimers[event.id] = setTimeout(() => {
      const payload = { ...event, ownerUid: currentUser.uid };
      firestore.collection('events').doc(event.id).set(payload, { merge: true })
        .catch(err => console.warn('클라우드 저장 실패(로컬엔 이미 반영됨):', err));
    }, 800);
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

  async function bootstrapApp() {
    loadLocalCache();
    router(); // 로컬 캐시로 즉시 첫 렌더 — 로딩 화면 없음

    if (!initFirebase()) return; // 플레이스홀더 상태면 완전한 로컬 모드로 끝

    auth.onAuthStateChanged(async (user) => {
      const wasLoggedIn = !!currentUser;
      currentUser = user;
      if (user) {
        await syncFromCloud(user);
      } else if (wasLoggedIn) {
        loadLocalCache(); // 로그아웃: 캐시를 로컬 상태로 되돌림
      }
      router(); // 현재 화면 재렌더(로그인 버튼 상태 갱신 포함)
    });
  }
```

**중요**: 새 `store`는 더 이상 `_readAll`/`_writeAll` 메서드를 갖지 않는다(`_readAllLocal`/`_writeAllLocal`로 이름이 바뀌었다). 파일 전체에서 `store._readAll`/`store._writeAll`을 호출하는 다른 코드가 없는지 `grep -n "store\._readAll\|store\._writeAll" index.html`로 확인한다 — Stage 1~3 어디에도 `store`의 내부 메서드를 외부에서 직접 호출하는 코드는 없었으므로(모두 `store.getAll/get/save/remove`만 사용) 이 교체는 안전해야 한다. 만약 예상 밖의 호출부가 발견되면 BLOCKED로 보고한다.

- [ ] **Step 2: `runRegressionTests`를 `async`로 전환하고 store/동기화 회귀 테스트 추가**

Find:
```javascript
  function runRegressionTests() {
```

Replace:
```javascript
  async function runRegressionTests() {
```

Find(회귀 테스트 마지막 블록 — `photoStore: usage()는 숫자 반환` assertion부터 `전 테스트 공통` 주석 사이):
```javascript
    assert('photoStore: usage()는 숫자 반환', typeof photoStore.usage() === 'number');

    // ---- 전 테스트 공통: Σ분배액 === amount ----
```

Replace:
```javascript
    assert('photoStore: usage()는 숫자 반환', typeof photoStore.usage() === 'number');

    // ---- store: 저장 시 shareId 자동 부여 ----
    const evShare = createEvent('shareId 테스트', '2026-01-01');
    addMember(evShare, '총무');
    store.save(evShare);
    assert('store.save: shareId 20자 자동 생성', typeof evShare.shareId === 'string' && evShare.shareId.length === 20);
    store.remove(evShare.id);

    // ---- syncFromCloud: 마이그레이션은 로컬 전용 이벤트만 업로드, 기존 클라우드 데이터는 절대 덮어쓰지 않음 ----
    function makeFakeFirestore(initialDocs) {
      const docs = new Map(initialDocs.map(d => [d.id, d]));
      function queryFrom(list) {
        return {
          where(field, op, value) { return queryFrom(list.filter(d => d[field] === value)); },
          limit(n) { return queryFrom(list.slice(0, n)); },
          get: () => Promise.resolve({ docs: list.map(d => ({ data: () => d })), empty: list.length === 0 }),
        };
      }
      return {
        collection() {
          return Object.assign(queryFrom(Array.from(docs.values())), {
            doc(id) {
              return {
                set: (payload) => { docs.set(id, { ...payload, id }); return Promise.resolve(); },
                delete: () => { docs.delete(id); return Promise.resolve(); },
              };
            },
          });
        },
        _docs: docs,
      };
    }

    const cloudOnlyEvent = { id: 'cloud-only-1', name: '클라우드 전용', ownerUid: 'test-uid', shareId: 'existing-share-id-0001', members: [], items: [] };
    const localOnlyEvent = createEvent('로컬 전용', '2026-01-01');
    addMember(localOnlyEvent, '총무');

    const savedFirestore = firestore, savedCurrentUser = currentUser, savedFirebaseReady = firebaseReady;
    const fakeFs = makeFakeFirestore([cloudOnlyEvent]);
    firestore = fakeFs;
    firebaseReady = true;
    eventsCache.push(localOnlyEvent);
    localStorage.removeItem(CONFIG.MIGRATION_FLAG_PREFIX + 'test-uid');

    await syncFromCloud({ uid: 'test-uid' });

    assert('syncFromCloud: 로컬 전용 이벤트가 클라우드로 업로드됨', fakeFs._docs.has(localOnlyEvent.id));
    assert('syncFromCloud: 기존 클라우드 이벤트는 덮어쓰이지 않고 그대로 유지됨', fakeFs._docs.get('cloud-only-1').name === '클라우드 전용');
    assert('syncFromCloud: 병합 후 캐시에 두 이벤트 모두 포함', store.get('cloud-only-1') !== null && store.get(localOnlyEvent.id) !== null);
    assert('syncFromCloud: 마이그레이션 플래그 기록됨', localStorage.getItem(CONFIG.MIGRATION_FLAG_PREFIX + 'test-uid') === '1');

    fakeFs._docs.delete(localOnlyEvent.id); // 클라우드에서만 제거해 재업로드 여부를 관찰
    eventsCache = [localOnlyEvent];
    await syncFromCloud({ uid: 'test-uid' });
    assert('syncFromCloud: 마이그레이션 플래그가 있으면 재업로드하지 않음', !fakeFs._docs.has(localOnlyEvent.id));

    localStorage.removeItem(CONFIG.MIGRATION_FLAG_PREFIX + 'test-uid');
    firestore = savedFirestore;
    currentUser = savedCurrentUser;
    firebaseReady = savedFirebaseReady;
    loadLocalCache(); // 테스트로 오염된 eventsCache를 실제 localStorage 상태로 복구

    // ---- 전 테스트 공통: Σ분배액 === amount ----
```

`makeFakeFirestore`는 이후 Task 5(공유 뷰 화면 테스트)에서도 재사용된다 — 이름과 시그니처(`makeFakeFirestore(initialDocs)`, 반환 객체가 `collection()`으로 `where/limit/get`과 `doc(id).set/delete`를 모두 지원)를 그대로 유지할 것.

- [ ] **Step 3: `DOMContentLoaded` 리스너를 `bootstrapApp`으로 교체**

Find:
```javascript
  window.addEventListener('DOMContentLoaded', router);
  if (CONFIG.DEV_MODE) {
    window.addEventListener('DOMContentLoaded', runRegressionTests);
  }
```

Replace:
```javascript
  window.addEventListener('DOMContentLoaded', bootstrapApp);
  if (CONFIG.DEV_MODE) {
    window.addEventListener('DOMContentLoaded', runRegressionTests);
  }
```

(등록 순서를 유지한다 — `bootstrapApp`이 먼저 실행되어 `eventsCache`를 채운 뒤에 회귀 테스트가 실행되어야 한다. `CONFIG.FIREBASE.apiKey`가 빈 값인 커밋 상태에서는 `bootstrapApp`이 동기 부분(`loadLocalCache()`+`router()`)만 실행하고 `initFirebase()`가 `false`를 반환하며 즉시 끝나므로, 두 리스너의 실행 순서 문제는 실질적으로 발생하지 않는다.)

- [ ] **Step 4: 검증**

Node + jsdom으로 `index.html`을 로드하고(`CONFIG.DEV_MODE = true`) `runRegressionTests()`를 `await`로 실행 — 기존 49개 + 신규 assertion(6개) = 55개 전부 pass 확인. `window.eval`로 실행할 때 `runRegressionTests`가 이제 `async function`이므로 반환값이 Promise임을 감안해 `await win.eval('runRegressionTests()')` 형태로 호출해야 한다(단순 `win.eval(...)`는 Promise 자체를 반환하므로 그 뒤에 `.then` 또는 `await`가 필요 — jsdom eval 컨텍스트에서 top-level await가 안 되면 `win.eval('(async () => { window.__testResults = await runRegressionTests(); })()')` 후 `win.__testResults`를 읽는 방식을 쓴다).

또한 `runSelfTests()`(Stage 1의 기존 수동 테스트 함수, `store.save/get/remove`를 직접 호출)도 여전히 정상 동작하는지 별도로 확인한다 — 이 함수는 `async`로 바꾸지 않는다(모두 동기 호출만 사용하므로 변경 불필요).

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "Stage 4: store 메모리 캐시 재구현 + bootstrapApp + 로그인 시 마이그레이션 동기화"
```

---

## Task 3: 홈 화면 로그인/로그아웃 버튼

**Files:**
- Modify: `index.html:1362-1389`(`renderHome` 함수)

**Interfaces:**
- Consumes: `firebaseReady`, `currentUser`, `signInWithGoogle()`, `signOutUser()`(Task 1), `renderBadge`(기존)
- Produces: `renderAuthButton()`(순수 문자열 반환 함수, 다른 태스크는 소비하지 않음 — 홈 화면 전용)

- [ ] **Step 1: `renderAuthButton()` 추가 + `renderHome`의 topbar에 삽입**

Find:
```javascript
  function renderHome() {
    const app = document.getElementById('app');
    const events = store.getAll().slice().sort((a, b) => b.date.localeCompare(a.date));

    app.innerHTML = `
      <header class="topbar">
        <h1>${escapeHtml(CONFIG.APP_SHORT)}</h1>
      </header>
      <main class="screen">
        ${events.length === 0 ? renderEmptyState() : `<div class="event-list">${events.map(renderEventCard).join('')}</div>`}
      </main>
      <button type="button" id="btn-new-event" class="fab">+ 새 모임</button>
    `;

    document.getElementById('btn-new-event').addEventListener('click', onCreateEvent);
    app.querySelectorAll('.event-card').forEach(card => {
      card.addEventListener('click', (e) => {
        if (e.target.closest('.card-menu-btn')) return;
        location.hash = `#/event/${card.dataset.id}`;
      });
    });
    app.querySelectorAll('.card-menu-btn').forEach(btn => {
      btn.addEventListener('click', (e) => {
        e.stopPropagation();
        onDeleteEvent(btn.dataset.id);
      });
    });
  }
```

Replace:
```javascript
  function renderAuthButton() {
    if (!firebaseReady) {
      return `<button type="button" class="btn btn-ghost" disabled title="Firebase 설정이 필요해요">로그인</button>`;
    }
    if (currentUser) {
      return `<button type="button" id="btn-logout" class="btn btn-ghost">${escapeHtml(currentUser.displayName || '사용자')} · 로그아웃</button>`;
    }
    return `<button type="button" id="btn-login" class="btn btn-ghost">Google로 로그인</button>`;
  }

  function renderHome() {
    const app = document.getElementById('app');
    const events = store.getAll().slice().sort((a, b) => b.date.localeCompare(a.date));

    app.innerHTML = `
      <header class="topbar">
        <h1>${escapeHtml(CONFIG.APP_SHORT)}</h1>
        ${renderAuthButton()}
      </header>
      <main class="screen">
        ${events.length === 0 ? renderEmptyState() : `<div class="event-list">${events.map(renderEventCard).join('')}</div>`}
      </main>
      <button type="button" id="btn-new-event" class="fab">+ 새 모임</button>
    `;

    document.getElementById('btn-new-event').addEventListener('click', onCreateEvent);
    app.querySelectorAll('.event-card').forEach(card => {
      card.addEventListener('click', (e) => {
        if (e.target.closest('.card-menu-btn')) return;
        location.hash = `#/event/${card.dataset.id}`;
      });
    });
    app.querySelectorAll('.card-menu-btn').forEach(btn => {
      btn.addEventListener('click', (e) => {
        e.stopPropagation();
        onDeleteEvent(btn.dataset.id);
      });
    });

    const loginBtn = document.getElementById('btn-login');
    if (loginBtn) loginBtn.addEventListener('click', signInWithGoogle);
    const logoutBtn = document.getElementById('btn-logout');
    if (logoutBtn) logoutBtn.addEventListener('click', signOutUser);
  }
```

(`.topbar h1 { flex: 1 }`이 이미 CSS에 있으므로 `<h1>` 다음에 오는 버튼은 자동으로 우측 정렬된다 — 별도 CSS 추가 불필요.)

- [ ] **Step 2: 검증**

jsdom으로 `renderHome()`을 호출해 다음 3가지 상태를 각각 확인한다(모듈 스코프 `firebaseReady`/`currentUser`를 테스트에서 직접 대입해 전환):
1. `firebaseReady = false` → `#btn-login`/`#btn-logout` 없음, 비활성 "로그인" 버튼 텍스트 존재
2. `firebaseReady = true; currentUser = null;` → `#btn-login` 존재, 클릭 시 `signInWithGoogle` 호출됨(스파이/카운터로 확인)
3. `firebaseReady = true; currentUser = { uid: 'u1', displayName: '테스트유저' };` → `#btn-logout` 존재, 텍스트에 "테스트유저" 포함, 클릭 시 `signOutUser` 호출됨

테스트 후 `firebaseReady = false; currentUser = null;`로 복구한다. 기존 회귀 테스트(`runRegressionTests`)에는 추가하지 않는다(이건 UI 렌더링 테스트이지 도메인 로직 회귀가 아니므로 별도 스크립트로 충분) — 단, jsdom 검증 결과는 리포트에 남긴다.

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "Stage 4: 홈 화면 Google 로그인/로그아웃 버튼"
```

---

## Task 4: 정산 표·송금 카드 추출 리팩터링 + 공유 링크 복사 버튼

**Files:**
- Modify: `index.html:1859-1958`(`renderSettlementResult` 전체를 두 헬퍼 함수 추출 후 축소된 버전으로 교체)

**Interfaces:**
- Consumes: `calcSettlement`(기존), `currentUser`(Task 2), `escapeHtml`(기존)
- Produces: `renderSettlementTable(ev, result)`, `renderTransferCard(ev, result)` — Task 5(`renderShareView`)가 그대로 재사용한다. 시그니처를 바꾸지 말 것.

- [ ] **Step 1: `renderSettlementResult` 전체를 교체**

Find(현재 함수 전체 — `function renderSettlementResult(params) {`부터 그 함수의 닫는 `}`까지):
```javascript
  function renderSettlementResult(params) {
    const app = document.getElementById('app');
    const ev = store.get(params.id);
    if (!ev) { location.hash = '#/'; return; }

    const result = calcSettlement(ev);
    const sortedItems = ev.items.slice().sort((a, b) => a.order - b.order);
    const owner = ev.members.find(m => m.isOwner);
    const hasAccount = ev.account.bank || ev.account.number || ev.account.holder;

    app.innerHTML = `
      <header class="topbar">
        <button type="button" class="btn-back" id="btn-back">←</button>
        <h1>정산 결과</h1>
      </header>
      <main class="screen">
        ${result.errors.length > 0 ? `<div class="card-section field-error">${result.errors.map(e => escapeHtml(e.message)).join('<br>')}</div>` : ''}
        <div class="settlement-table-wrap">
          <table class="settlement-table">
            <thead>
              <tr>
                <th>항목</th>
                ${ev.members.map(m => `<th>${m.emoji}<br><span class="settlement-th-name">${escapeHtml(m.name)}</span></th>`).join('')}
                <th>합계</th>
              </tr>
            </thead>
            <tbody>
              ${sortedItems.map(item => `
                <tr>
                  <td>${item.emoji ? escapeHtml(item.emoji) + ' ' : ''}${escapeHtml(item.name)}</td>
                  ${ev.members.map(m => {
                    const value = result.matrix[item.id] ? result.matrix[item.id][m.id] : undefined;
                    return `<td>${value === undefined ? '<span class="settlement-cell-empty">—</span>' : '₩' + value.toLocaleString(CONFIG.CURRENCY_LOCALE)}</td>`;
                  }).join('')}
                  <td>₩${result.itemTotals[item.id].toLocaleString(CONFIG.CURRENCY_LOCALE)}</td>
                </tr>
              `).join('')}
              <tr class="settlement-total-row">
                <th>총계</th>
                ${ev.members.map(m => `<td>₩${(result.memberTotals[m.id] || 0).toLocaleString(CONFIG.CURRENCY_LOCALE)}</td>`).join('')}
                <td>₩${result.grandTotal.toLocaleString(CONFIG.CURRENCY_LOCALE)}</td>
              </tr>
            </tbody>
          </table>
        </div>

        <section class="card-section">
          <h2>${owner ? escapeHtml(owner.name) : '총무'}에게 보낼 금액</h2>
          <div class="transfer-card">
            ${ev.members.map(m => {
              if (owner && m.id === owner.id) {
                return `<div class="transfer-row transfer-owner-row"><span>${m.emoji} ${escapeHtml(m.name)}</span><span>결제 완료 🧾</span></div>`;
              }
              return `<div class="transfer-row"><span>${m.emoji} ${escapeHtml(m.name)}</span><span class="transfer-amount">₩${(result.transfers[m.id] || 0).toLocaleString(CONFIG.CURRENCY_LOCALE)}</span></div>`;
            }).join('')}
          </div>
          ${hasAccount ? `
            <div class="account-info-box">
              <span>${escapeHtml(ev.account.bank)} ${escapeHtml(ev.account.number)} ${escapeHtml(ev.account.holder)}</span>
              <button type="button" id="btn-copy-account" class="btn btn-ghost">복사</button>
            </div>
          ` : ''}
        </section>

        <section class="card-section">
          <button type="button" id="btn-settle" class="btn ${(ev.status === 'settled' || result.errors.length > 0) ? 'btn-disabled' : 'btn-primary'}" ${(ev.status === 'settled' || result.errors.length > 0) ? 'disabled' : ''}>
            ${ev.status === 'settled' ? '정산 완료됨 ✓' : '정산 완료'}
          </button>
          ${result.errors.length > 0 ? '<div class="field-hint">오류가 있는 항목을 먼저 수정해주세요</div>' : ''}
        </section>

        <section class="card-section">
          <h2>멤버별 영수증</h2>
          <div class="member-list">
            ${ev.members.map(m => `<button type="button" class="btn btn-ghost btn-receipt" data-id="${m.id}" style="width:100%">${m.emoji} ${escapeHtml(m.name)} 영수증 보기</button>`).join('')}
          </div>
        </section>
      </main>
    `;

    document.getElementById('btn-back').addEventListener('click', () => { location.hash = `#/event/${ev.id}/items`; });

    const copyBtn = document.getElementById('btn-copy-account');
    if (copyBtn) {
      copyBtn.addEventListener('click', () => {
        const text = `${ev.account.bank} ${ev.account.number} ${ev.account.holder}`.trim();
        navigator.clipboard.writeText(text).then(() => showToast('계좌 정보를 복사했어요'));
      });
    }

    if (result.errors.length === 0 && ev.status !== 'settled') {
      document.getElementById('btn-settle').addEventListener('click', () => {
        ev.status = 'settled';
        store.save(ev);
        renderSettlementResult(params);
      });
    }

    document.querySelectorAll('.btn-receipt').forEach(btn => {
      btn.addEventListener('click', () => { location.hash = `#/event/${ev.id}/receipt/${btn.dataset.id}`; });
    });
  }
```

Replace:
```javascript
  function renderSettlementTable(ev, result) {
    const sortedItems = ev.items.slice().sort((a, b) => a.order - b.order);
    return `
      <div class="settlement-table-wrap">
        <table class="settlement-table">
          <thead>
            <tr>
              <th>항목</th>
              ${ev.members.map(m => `<th>${m.emoji}<br><span class="settlement-th-name">${escapeHtml(m.name)}</span></th>`).join('')}
              <th>합계</th>
            </tr>
          </thead>
          <tbody>
            ${sortedItems.map(item => `
              <tr>
                <td>${item.emoji ? escapeHtml(item.emoji) + ' ' : ''}${escapeHtml(item.name)}</td>
                ${ev.members.map(m => {
                  const value = result.matrix[item.id] ? result.matrix[item.id][m.id] : undefined;
                  return `<td>${value === undefined ? '<span class="settlement-cell-empty">—</span>' : '₩' + value.toLocaleString(CONFIG.CURRENCY_LOCALE)}</td>`;
                }).join('')}
                <td>₩${result.itemTotals[item.id].toLocaleString(CONFIG.CURRENCY_LOCALE)}</td>
              </tr>
            `).join('')}
            <tr class="settlement-total-row">
              <th>총계</th>
              ${ev.members.map(m => `<td>₩${(result.memberTotals[m.id] || 0).toLocaleString(CONFIG.CURRENCY_LOCALE)}</td>`).join('')}
              <td>₩${result.grandTotal.toLocaleString(CONFIG.CURRENCY_LOCALE)}</td>
            </tr>
          </tbody>
        </table>
      </div>
    `;
  }

  function renderTransferCard(ev, result) {
    const owner = ev.members.find(m => m.isOwner);
    const hasAccount = ev.account.bank || ev.account.number || ev.account.holder;
    return `
      <section class="card-section">
        <h2>${owner ? escapeHtml(owner.name) : '총무'}에게 보낼 금액</h2>
        <div class="transfer-card">
          ${ev.members.map(m => {
            if (owner && m.id === owner.id) {
              return `<div class="transfer-row transfer-owner-row"><span>${m.emoji} ${escapeHtml(m.name)}</span><span>결제 완료 🧾</span></div>`;
            }
            return `<div class="transfer-row"><span>${m.emoji} ${escapeHtml(m.name)}</span><span class="transfer-amount">₩${(result.transfers[m.id] || 0).toLocaleString(CONFIG.CURRENCY_LOCALE)}</span></div>`;
          }).join('')}
        </div>
        ${hasAccount ? `
          <div class="account-info-box">
            <span>${escapeHtml(ev.account.bank)} ${escapeHtml(ev.account.number)} ${escapeHtml(ev.account.holder)}</span>
            <button type="button" id="btn-copy-account" class="btn btn-ghost">복사</button>
          </div>
        ` : ''}
      </section>
    `;
  }

  function renderSettlementResult(params) {
    const app = document.getElementById('app');
    const ev = store.get(params.id);
    if (!ev) { location.hash = '#/'; return; }

    const result = calcSettlement(ev);

    app.innerHTML = `
      <header class="topbar">
        <button type="button" class="btn-back" id="btn-back">←</button>
        <h1>정산 결과</h1>
      </header>
      <main class="screen">
        ${result.errors.length > 0 ? `<div class="card-section field-error">${result.errors.map(e => escapeHtml(e.message)).join('<br>')}</div>` : ''}
        ${renderSettlementTable(ev, result)}
        ${renderTransferCard(ev, result)}
        ${currentUser && ev.ownerUid
          ? `<button type="button" id="btn-copy-share-link" class="btn btn-ghost">🔗 공유 링크 복사</button>`
          : `<div class="field-hint">로그인하면 공유 링크를 만들 수 있어요</div>`}

        <section class="card-section">
          <button type="button" id="btn-settle" class="btn ${(ev.status === 'settled' || result.errors.length > 0) ? 'btn-disabled' : 'btn-primary'}" ${(ev.status === 'settled' || result.errors.length > 0) ? 'disabled' : ''}>
            ${ev.status === 'settled' ? '정산 완료됨 ✓' : '정산 완료'}
          </button>
          ${result.errors.length > 0 ? '<div class="field-hint">오류가 있는 항목을 먼저 수정해주세요</div>' : ''}
        </section>

        <section class="card-section">
          <h2>멤버별 영수증</h2>
          <div class="member-list">
            ${ev.members.map(m => `<button type="button" class="btn btn-ghost btn-receipt" data-id="${m.id}" style="width:100%">${m.emoji} ${escapeHtml(m.name)} 영수증 보기</button>`).join('')}
          </div>
        </section>
      </main>
    `;

    document.getElementById('btn-back').addEventListener('click', () => { location.hash = `#/event/${ev.id}/items`; });

    const copyBtn = document.getElementById('btn-copy-account');
    if (copyBtn) {
      copyBtn.addEventListener('click', () => {
        const text = `${ev.account.bank} ${ev.account.number} ${ev.account.holder}`.trim();
        navigator.clipboard.writeText(text).then(() => showToast('계좌 정보를 복사했어요'));
      });
    }

    const shareLinkBtn = document.getElementById('btn-copy-share-link');
    if (shareLinkBtn) {
      shareLinkBtn.addEventListener('click', () => {
        const url = `${location.origin}${location.pathname}#/view/${ev.shareId}`;
        navigator.clipboard.writeText(url).then(() => showToast('공유 링크를 복사했어요'));
      });
    }

    if (result.errors.length === 0 && ev.status !== 'settled') {
      document.getElementById('btn-settle').addEventListener('click', () => {
        ev.status = 'settled';
        store.save(ev);
        renderSettlementResult(params);
      });
    }

    document.querySelectorAll('.btn-receipt').forEach(btn => {
      btn.addEventListener('click', () => { location.hash = `#/event/${ev.id}/receipt/${btn.dataset.id}`; });
    });
  }
```

- [ ] **Step 2: 검증**

jsdom으로 시드 A 이벤트를 만들어 `renderSettlementTable(ev, result)`/`renderTransferCard(ev, result)`를 직접 호출 — 반환된 HTML 문자열에 기존과 동일한 표/카드 마크업이 그대로 나오는지 확인(리팩터링이므로 출력이 한 글자도 달라지면 안 된다 — 교체 전후 `renderSettlementResult(params)` 전체 출력 문자열을 diff해 표/카드 부분이 100% 동일함을 확인하는 방식을 권장). 그다음 `currentUser`/`ev.ownerUid` 조합 2가지(둘 다 있음 → 공유 링크 버튼 노출, 없음 → 안내 문구만 노출)를 jsdom으로 확인. 마지막으로 `runRegressionTests()`를 재실행해 55개 전부 pass 확인(이 리팩터링은 회귀 테스트 로직과 무관하지만 무회귀 확인 차원).

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "Stage 4: 정산 표·송금 카드를 renderSettlementTable/renderTransferCard로 추출 + 공유 링크 복사 버튼"
```

---

## Task 5: 공유 뷰 화면 (`#/view/{shareId}`)

**Files:**
- Modify: `index.html`(`matchRoute` 함수, 라우트 등록부, 신규 `renderShareView` 함수 추가)

**Interfaces:**
- Consumes: `renderSettlementTable`/`renderTransferCard`(Task 4), `calcSettlement`(기존), `firebaseReady`/`firestore`(Task 1/2), `renderBadge`(기존), `makeFakeFirestore`(Task 2가 회귀 테스트 안에 정의한 헬퍼 — 이 태스크의 검증에서 그대로 재사용)
- Produces: `renderShareView(params)` — 라우터에서만 호출, 다른 태스크는 소비하지 않음.

- [ ] **Step 1: `matchRoute`에 `/view/:shareId` 패턴 추가**

Find:
```javascript
    const receiptMatch = hash.match(/^\/event\/([^/]+)\/receipt\/([^/]+)$/);
    if (receiptMatch && routes['/event/:id/receipt/:memberId']) {
      return { render: routes['/event/:id/receipt/:memberId'], params: { id: receiptMatch[1], memberId: receiptMatch[2] } };
    }
    return null;
  }
```

Replace:
```javascript
    const receiptMatch = hash.match(/^\/event\/([^/]+)\/receipt\/([^/]+)$/);
    if (receiptMatch && routes['/event/:id/receipt/:memberId']) {
      return { render: routes['/event/:id/receipt/:memberId'], params: { id: receiptMatch[1], memberId: receiptMatch[2] } };
    }
    const viewMatch = hash.match(/^\/view\/([^/]+)$/);
    if (viewMatch && routes['/view/:shareId']) {
      return { render: routes['/view/:shareId'], params: { shareId: viewMatch[1] } };
    }
    return null;
  }
```

- [ ] **Step 2: `renderShareView` 함수 추가**

`renderSettlementResult`/`renderSettlementTable`/`renderTransferCard` 정의 바로 다음(index.html의 정산 결과 화면 섹션 끝)에 추가:

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
        ${renderBadge('🔒 읽기 전용', 'default')}
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

- [ ] **Step 3: 라우트 등록**

Find:
```javascript
  registerRoute('/event/:id/receipt/:memberId', renderReceiptView);
```

Replace:
```javascript
  registerRoute('/event/:id/receipt/:memberId', renderReceiptView);
  registerRoute('/view/:shareId', renderShareView);
```

- [ ] **Step 4: 검증**

jsdom + `runRegressionTests`의 `makeFakeFirestore` 헬퍼를 재사용해(같은 파일 안에 이미 정의돼 있으므로 새로 정의하지 않고 `window.eval`로 그 함수 정의를 먼저 가져오거나, 별도 테스트 스크립트에서 동일한 헬퍼를 그대로 복사해 사용) 3가지 경로를 확인:
1. `firebaseReady = false` 상태로 `renderShareView({ shareId: 'x' })` 호출 → "공유 기능이 아직 설정되지 않았어요" 노출
2. `firebaseReady = true`, `firestore = makeFakeFirestore([])`(빈 배열) → `renderShareView({ shareId: 'no-such-id' })` 호출 → "모임을 찾을 수 없어요" 노출
3. `firebaseReady = true`, `firestore = makeFakeFirestore([시드 A 이벤트 객체(shareId: 'abc123' 포함)])` → `renderShareView({ shareId: 'abc123' })` 호출 → `.settlement-table`/`.transfer-card` 존재, `정산 완료`/`영수증 보기`/`계좌 입력` 등 편집 전용 요소는 전혀 없음(`document.getElementById('btn-settle')`이 `null`인지 등으로 확인)

테스트 후 `firebaseReady = false; firestore = null;`로 복구한다.

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "Stage 4: 공유 뷰 화면(#/view/:shareId) — 비로그인 읽기 전용 정산 결과"
```

---

## Task 6: PWA manifest·아이콘·계좌 안내 문구

**Files:**
- Create: `icons/icon.svg`
- Create: `manifest.json`
- Modify: `index.html`(`<head>` 링크 4개 추가, 계좌 정보 필드에 안내 문구 추가)

**Interfaces:**
- Consumes: 없음(정적 리소스 + 마크업 추가뿐)
- Produces: 없음(다른 태스크가 소비하지 않는 마감 작업)

- [ ] **Step 1: `icons/icon.svg` 생성**

```xml
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512">
  <rect width="512" height="512" rx="96" fill="#FF7A45"/>
  <text x="256" y="336" font-size="260" text-anchor="middle">🧾</text>
</svg>
```

- [ ] **Step 2: `manifest.json` 생성**

```json
{
  "name": "오늘은 내가 총무",
  "short_name": "오.내.총",
  "start_url": "./index.html",
  "display": "standalone",
  "background_color": "#FFF9F2",
  "theme_color": "#FF7A45",
  "icons": [
    { "src": "icons/icon.svg", "sizes": "any", "type": "image/svg+xml", "purpose": "any" }
  ]
}
```

이 값들(`name`/`short_name`/`background_color`/`theme_color`)은 `index.html`의 `CONFIG.APP_NAME`(`'오늘은 내가 총무'`)/`CONFIG.APP_SHORT`(`'오.내.총'`)/`CONFIG.COLOR_BG`(`'#FFF9F2'`)/`CONFIG.COLOR_PRIMARY`(`'#FF7A45'`)와 반드시 동일해야 한다(정적 JSON이라 JS CONFIG를 참조할 수 없으므로 수동으로 값을 맞춘다) — 구현 전 `index.html`에서 그 4개 CONFIG 값을 다시 확인하고 정확히 일치시킬 것.

- [ ] **Step 3: `<head>`에 manifest/아이콘 링크 추가**

Find:
```html
<title>오늘은 내가 총무</title>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/static/pretendard.css" />
<style>
```

Replace:
```html
<title>오늘은 내가 총무</title>
<!-- 아래 4개 값은 manifest.json 및 CONFIG.APP_NAME/APP_SHORT/COLOR_BG/COLOR_PRIMARY와 반드시 일치시킬 것 -->
<link rel="manifest" href="manifest.json">
<link rel="apple-touch-icon" href="icons/icon.svg">
<link rel="icon" href="icons/icon.svg" type="image/svg+xml">
<meta name="theme-color" content="#FF7A45">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/static/pretendard.css" />
<style>
```

- [ ] **Step 4: 계좌 정보 필드에 안내 문구 추가**

Find:
```html
          <label class="field-label">계좌 정보 (선택)</label>
          <div class="account-fields">
            <input id="input-bank" class="input" type="text" placeholder="은행" value="${escapeAttr(ev.account.bank)}" />
            <input id="input-number" class="input" type="text" placeholder="계좌번호" value="${escapeAttr(ev.account.number)}" />
            <input id="input-holder" class="input" type="text" placeholder="예금주" value="${escapeAttr(ev.account.holder)}" />
          </div>
        </section>
```

Replace:
```html
          <label class="field-label">계좌 정보 (선택)</label>
          <div class="account-fields">
            <input id="input-bank" class="input" type="text" placeholder="은행" value="${escapeAttr(ev.account.bank)}" />
            <input id="input-number" class="input" type="text" placeholder="계좌번호" value="${escapeAttr(ev.account.number)}" />
            <input id="input-holder" class="input" type="text" placeholder="예금주" value="${escapeAttr(ev.account.holder)}" />
          </div>
          <div class="field-hint">공유 링크와 영수증에 노출됩니다</div>
        </section>
```

- [ ] **Step 5: 검증**

jsdom으로 `document.head.querySelector('link[rel="manifest"]')` 등 4개 링크 태그가 정확한 `href`/속성으로 존재하는지 확인. `manifest.json`을 `JSON.parse(fs.readFileSync(...))`로 파싱해 문법 오류 없음과 4개 값이 `index.html`의 CONFIG 값과 문자열 그대로 일치하는지 확인. `icons/icon.svg`도 `fs.readFileSync`로 읽어 `<svg`로 시작하는 유효한 XML인지 확인(완전한 XML 파서는 필요 없음 — 기본적인 well-formed 여부만). `renderEventSettings`를 호출해 반환 HTML에 "공유 링크와 영수증에 노출됩니다" 문자열이 포함되는지 확인.

Chrome 브라우저 자동화가 가능하면 실제로 `manifest.json`이 브라우저에서 파싱 오류 없이 로드되는지, `icons/icon.svg`가 시각적으로 정상 렌더링되는지(둥근 사각형 배경 + 🧾 이모지가 잘리지 않고 중앙에 보이는지) 스크린샷으로 확인하고, 필요하면 `icon.svg`의 `font-size`/`y` 값을 미세 조정한다.

- [ ] **Step 6: 커밋**

```bash
git add index.html manifest.json icons/icon.svg
git commit -m "Stage 5: PWA manifest·아이콘 + 계좌 정보 노출 안내 문구"
```

---

## Task 7: 완료 기준 검증 + 375px 반응형 실브라우저 점검 + CLAUDE.md 갱신

**Files:**
- Modify: `CLAUDE.md`
- (필요 시) Modify: `index.html` — 375px 점검 중 발견된 실제 레이아웃 깨짐 수정(새 기능 추가 아님, 기존 CSS 미세 조정에 한정)

**Interfaces:**
- Consumes: 전체 Stage 4+5 구현(Task 1-6)
- Produces: 없음(마감 태스크)

- [ ] **Step 1: 완료 기준 검증**

이번 세션에는 실제 Firebase 프로젝트가 없다(사용자가 이후 직접 생성) — 검증은 3가지로 나뉜다:

**(A) 실브라우저로 완전히 검증 가능** (Chrome 자동화 시도, 안 되면 jsdom으로 대체하고 "실브라우저 필요"로 명시):
1. 비로그인 상태에서 홈→모임 생성→멤버 등록→결제 입력→정산 결과까지 기존과 동일하게 전부 동작(무회귀) — Stage 2/3 수준의 실제 UI 조작으로 확인
2. `initFirebase()`가 `CONFIG.FIREBASE.apiKey`가 빈 값인 현재 커밋 상태에서 안전하게 스킵되는지(콘솔에 Firebase 관련 에러 없음, 홈 화면에 비활성 "로그인" 버튼만 노출)
3. `manifest.json`/`icons/icon.svg` 정상 로드(Task 6에서 이미 확인했다면 재확인만)
4. 375px 뷰포트에서 6개 화면(홈, 모임 설정, 결제 입력, 정산 결과, 영수증, 공유 뷰 — 공유 뷰는 `firebaseReady=false` 상태의 안내 화면으로 확인) 스크린샷 — 실제로 잘리거나 겹치는 요소가 있으면 그 자리에서 CSS 수정

**(B) jsdom + 목(mock) Firestore로 검증 가능** (Task 2/5에서 이미 작성된 회귀 테스트를 재실행하는 것으로 충분 — 새로 작성할 필요 없음):
5. `store` 캐시·write-through 로직, `syncFromCloud` 마이그레이션 합집합 로직(기존 클라우드 데이터 보존) — `runRegressionTests()` 재실행으로 확인
6. `renderShareView`의 3가지 분기(미설정/못찾음/정상) — Task 5의 검증 재실행으로 확인

**(C) 이번 세션에 검증 불가능 — 코드 리뷰로만 확인, 사용자가 나중에 직접 실행**:
7. 실제 Google 로그인 팝업 → Firestore 문서 생성
8. 실제 로그인 상태에서 항목 수정 → 새로고침·타 브라우저 로그인에서 반영
9. `/#/view/{shareId}` 실데이터 조회
10. Security Rules가 실제로 미소유자 쓰기를 막는지

각 항목의 검증 방법과 결과를 최종 보고서에 명시한다. (A)/(B)에서 실제 버그를 발견하면 `index.html`을 수정하고 diff를 보고서에 남긴다.

- [ ] **Step 2: `CLAUDE.md` "Project status" 갱신**

Find:
```markdown
Stage 1 (skeleton: hash routing, CONFIG, event/member CRUD), Stage 2 (item entry, settlement engine, result table), and Stage 3 (감성 영수증: photo attach, dot-font receipt DOM, html2canvas capture/share) are implemented in `index.html`. Stages 4-5 (Firebase sync, PWA polish) are not yet implemented — see `오내총_PRD_v1.md` §9 and `docs/superpowers/specs/` for design docs, and `docs/superpowers/plans/` for implementation plans.
```

Replace:
```markdown
Stage 1 (skeleton: hash routing, CONFIG, event/member CRUD), Stage 2 (item entry, settlement engine, result table), Stage 3 (감성 영수증: photo attach, dot-font receipt DOM, html2canvas capture/share), and Stage 4+5 (Firebase Auth + Firestore sync, public share view, PWA manifest) are implemented in `index.html`. See `오내총_PRD_v1.md` §9 and `docs/superpowers/specs/` for design docs, and `docs/superpowers/plans/` for implementation plans.
```

- [ ] **Step 3: `CLAUDE.md`에 Firebase 동기화 구현 참고 섹션 추가**

Find:
```markdown
- `onDeleteEvent` calls `photoStore.removeByEvent(id)` immediately after `store.remove(id)` — deleting an event always cleans up its photos in the same action.

## Data storage model (hybrid — important, don't collapse into one store)
```

Replace:
```markdown
- `onDeleteEvent` calls `photoStore.removeByEvent(id)` immediately after `store.remove(id)` — deleting an event always cleans up its photos in the same action.

## Firebase sync & share view (Stage 4+5 implementation reference)

- `store` (in `index.html`) is a memory cache (`eventsCache`) with write-through persistence, not a direct Firestore/localStorage wrapper. `store.get/getAll/save/remove` remain fully synchronous — every call site elsewhere in the file is unaware of Firestore. `save()`/`remove()` always write `eventsCache` + `localStorage[CONFIG.STORAGE_KEY]` immediately; if a user is signed in (`currentUser` set, `firebaseReady === true`), they additionally schedule a debounced (800ms, per-event-id timer) best-effort Firestore write via `scheduleCloudSave` — a failed cloud write only logs `console.warn`, since the local write already succeeded.
- `initFirebase()` is the single gate: if `CONFIG.FIREBASE.apiKey` is empty (shipped default — real values are filled in by the project owner via the Firebase console), it returns `false` immediately without touching the `firebase` global at all, and the entire app runs in local-only mode exactly as in Stages 1-3. Never assume `firebase`/`auth`/`firestore` are defined without checking `firebaseReady` first.
- `bootstrapApp()` (registered on `DOMContentLoaded` in place of a bare `router()` call) always renders instantly from the local cache first, then — only if Firebase is configured — checks auth state and re-renders in the background once a cloud sync completes. There is no loading screen; a signed-in user may briefly see stale/local data before the cloud refresh lands.
- `syncFromCloud(user)` implements one-time-per-account migration: on first sync after login, any locally-cached event not already present in Firestore (by `id`) is uploaded; existing cloud documents for that `ownerUid` are never overwritten by local data. Completion is recorded in `localStorage[CONFIG.MIGRATION_FLAG_PREFIX + uid]` so it never re-runs for that account. This is a one-shot fetch, not an `onSnapshot` listener — sync reflects on refresh/re-login, not live within an open session.
- `shareId` (20-char, `crypto.randomUUID()`-derived) is assigned to every event the moment it's first saved, regardless of login state, so a locally-created event is already shareable once it syncs. `ownerUid` is only present on events that have been synced to Firestore at least once.
- The public share route `/#/view/{shareId}` (`renderShareView`) bypasses `store` entirely and queries Firestore directly (`where('shareId','==',...)`) — it works with no login and no local cache, by design. It reuses `renderSettlementTable`/`renderTransferCard` (also used by the owner's settlement result screen) but never `renderReceiptView` — the share view has no photos and no editing UI.
- Firestore Security Rules text lives in `firestore.rules.txt` (repo root) — it is not deployed by any script; the project owner pastes it into the Firebase console manually.

## Data storage model (hybrid — important, don't collapse into one store)
```

- [ ] **Step 4: `CLAUDE.md`의 데이터 저장 모델 표 갱신**

Find:
```markdown
| Events, members, items, settlement inputs, account info | **Firebase Firestore** | needs cross-device sync + shareable read links |
```

Replace:
```markdown
| Events, members, items, settlement inputs, account info | **Firebase Firestore when signed in, `localStorage` otherwise** (always mirrored to `localStorage` too — see "Firebase sync & share view" above) | needs cross-device sync + shareable read links, but must work with zero setup/no account |
```

- [ ] **Step 5: 커밋**

```bash
git add CLAUDE.md
# index.html은 375px 점검에서 실제 수정이 있었을 때만 추가
git commit -m "Stage 4+5: 완료 기준 검증 + CLAUDE.md Firebase 동기화 문서화"
```

- [ ] **Step 6: 사용자 체크리스트 정리**

코드 변경은 아니지만, 최종 보고서(태스크 리포트 및 코디네이터의 스테이지 종료 보고)에 사용자가 직접 해야 할 일을 순서대로 정리한다:
1. Firebase 콘솔(console.firebase.google.com)에서 새 프로젝트 생성
2. 프로젝트에 웹 앱 추가 → 발급된 설정값을 `index.html`의 `CONFIG.FIREBASE` 6개 필드에 채워넣기
3. Authentication → Google 로그인 방식 활성화
4. Firestore Database 생성(프로덕션 모드)
5. Firestore Rules 편집기에 `firestore.rules.txt` 내용 붙여넣기
6. Authentication → Settings → 승인된 도메인에 GitHub Pages 주소 추가
7. 배포 후 실제 로그인/동기화/공유 링크 동작을 수동으로 1회 확인(이번 세션에서 검증하지 못한 §11의 (C) 항목들)

---

## Self-Review 결과 (계획 작성자 자체 점검)

- **스펙 커버리지**: 브레인스토밍 4개 섹션(A~D, 스펙 §3~9) 전부 태스크로 매핑됨 — store 아키텍처(Task 2), 공유 뷰(Task 4-5), 인증·Security Rules(Task 1, 3), PWA 마감(Task 6). 스펙 §11 검증 전략은 Task 7에 반영. 스펙 §7 Security Rules 텍스트가 Task 1 Step 4와 정확히 일치.
- **placeholder 스캔**: "TBD"/"TODO" 없음. 모든 코드 스텝에 완전한 코드 포함.
- **타입/시그니처 일관성**: `store.getAll()/get(id)/save(event)/remove(id)` 시그니처가 Task 2 정의 이후 모든 소비 태스크(3,4,5,6)에서 동일하게 사용됨. `renderSettlementTable(ev, result)`/`renderTransferCard(ev, result)`가 Task 4 정의와 Task 5의 재사용에서 동일 시그니처. `makeFakeFirestore(initialDocs)`가 Task 2 정의와 Task 5의 재사용 설명에서 동일 이름·형태로 언급됨. `currentUser`/`firebaseReady`/`auth`/`firestore`가 Task 1에서 선언되고 Task 2~5에서 그대로 소비됨(재선언 없음).
- **의존성 순서**: Task 1(기반) → Task 2(store/부트스트랩, Task 1 소비) → Task 3(로그인 버튼 UI, Task 1+2 소비) → Task 4(리팩터링+공유버튼, Task 2의 `currentUser` 소비) → Task 5(공유 뷰, Task 4의 헬퍼 함수 소비) → Task 6(PWA, 독립) → Task 7(전체 검증). 역방향 의존 없음.
