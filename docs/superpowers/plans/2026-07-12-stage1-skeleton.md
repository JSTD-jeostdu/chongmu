# Stage 1 뼈대 구현 계획 (해시 라우팅 · CONFIG · 모임/멤버 CRUD)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 단일 `index.html`에 해시 라우팅, `CONFIG`, localStorage 기반 모임/멤버 CRUD를 구현하고, `docs/superpowers/specs/2026-07-12-stage1-skeleton-design.md`의 완료 기준 6개 항목을 모두 통과시킨다.

**Architecture:** 모든 코드는 `index.html` 한 파일 안의 `<style>`/`<script>` 블록에 들어간다. `<script>` 내부는 CONFIG → 유틸 → `store`(localStorage 추상화) → 순수 도메인 함수(모임/멤버 생성·검증) → 자체 테스트 함수 → 공용 UI 컴포넌트(토스트/다이얼로그/뱃지) → 해시 라우터 → 화면 렌더 함수 순서로 섹션을 나눠 추가한다. Firestore 이관(Stage 4)을 대비해 UI 코드는 `store`를 통해서만 데이터에 접근한다.

**Tech Stack:** HTML/CSS/Vanilla JS, Pretendard 웹폰트(CDN). 빌드 도구·npm·프레임워크 없음.

## Global Constraints

- 단일 `index.html` 파일 — HTML/CSS/JS를 별도 파일로 분리하지 않는다.
- 빌드 도구·npm·프레임워크 금지. 외부 리소스는 CDN만 허용.
- 모든 상수는 파일 상단 `CONFIG` 객체에 집약 — 매직 넘버/문자열 하드코딩 금지.
- 주석은 한국어, 섹션 구분 주석(`// ====== 섹션명 ======`)으로 코드 구조를 명확히 한다.
- 모바일 퍼스트 반응형: 기준 폭 390px, 최대 콘텐츠 폭 560px 중앙 정렬.
- 배포 대상 GitHub Pages — 상대 경로만 사용 (CDN 링크 제외).
- Stage 2~5 기능(정산 계산, 영수증, Firebase, PWA)은 이번 계획에서 구현하지 않는다.
- CSS 색상값은 하드코딩하지 않고 `CONFIG.COLOR_PRIMARY` / `CONFIG.COLOR_BG`를 JS에서 CSS 커스텀 프로퍼티로 주입해 사용한다.

## 검증 방법에 대한 메모

이 프로젝트는 npm/빌드 도구가 없는 단일 HTML이므로 Jest 등 테스트 프레임워크를 쓰지 않는다. 대신:
- **순수 로직(스토어, 도메인 함수)**: `index.html`에 포함된 `runSelfTests()` 함수가 브라우저 콘솔에서 assert 결과를 출력한다. 각 태스크의 "테스트 실행" 단계는 Chrome에서 `index.html`을 열고(`mcp__claude-in-chrome__navigate` + `javascript_tool`) `runSelfTests()`를 호출해 콘솔 출력을 확인하는 방식으로 수행한다.
- **UI/라우팅/반응형**: 같은 방식으로 Chrome을 열어 클릭·입력을 시뮬레이션(`computer` 도구) 하거나 `javascript_tool`로 DOM 상태를 확인한다.
- 파일 경로는 `file:///` URL로 열면 된다 (예: `file:///C:/Users/jeodu/Desktop/VibeCoding/오내총(오늘은내가총무)/index.html`).

---

### Task 1: HTML 뼈대 + CONFIG + 기본 레이아웃 CSS

**Files:**
- Create: `index.html`

**Interfaces:**
- Produces: 전역 `CONFIG` 객체 (아래 정확한 필드), `<div id="app">` 컨테이너, CSS 커스텀 프로퍼티 `--color-primary`, `--color-bg`.

- [ ] **Step 1: `index.html` 뼈대 작성**

```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>오늘은 내가 총무</title>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/static/pretendard.css" />
<style>
  /* ====== 리셋 & 레이아웃 ====== */
  * { box-sizing: border-box; }
  html, body {
    margin: 0;
    padding: 0;
    min-width: 390px;
    font-family: 'Pretendard', -apple-system, sans-serif;
    background: var(--color-bg, #FFF9F2);
    color: #3A2E28;
  }
  #app {
    max-width: 560px;
    margin: 0 auto;
    min-height: 100vh;
    padding: 0 16px 96px;
    position: relative;
  }
  h1, h2, p { margin: 0; }
  button { font-family: inherit; cursor: pointer; }
  input { font-family: inherit; }

  /* ====== 상단 바 ====== */
  .topbar {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 16px 0;
  }
  .topbar h1 {
    font-size: 20px;
    font-weight: 700;
    flex: 1;
  }

  /* ====== 공용 버튼 ====== */
  .btn {
    border: none;
    border-radius: 12px;
    padding: 12px 16px;
    font-size: 15px;
    font-weight: 600;
    min-height: 44px;
  }
  .btn-primary {
    background: var(--color-primary, #FF7A45);
    color: #fff;
  }
  .btn-ghost {
    background: #EFE6DC;
    color: #3A2E28;
  }
  .btn-danger {
    background: #E74C3C;
    color: #fff;
  }
  .btn-disabled {
    background: #E0D6CA;
    color: #9A8C7C;
  }

  .screen {
    display: flex;
    flex-direction: column;
    gap: 16px;
  }

  .card-section {
    background: #fff;
    border-radius: 16px;
    padding: 16px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.06);
    display: flex;
    flex-direction: column;
    gap: 10px;
  }
</style>
</head>
<body>
<div id="app"></div>

<script>
  // ====== CONFIG ======
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
    COLOR_PRIMARY: '#FF7A45',
    COLOR_BG: '#FFF9F2',
  };

  // CONFIG 색상값을 CSS 커스텀 프로퍼티로 주입 (CSS에 색상 하드코딩 금지)
  document.documentElement.style.setProperty('--color-primary', CONFIG.COLOR_PRIMARY);
  document.documentElement.style.setProperty('--color-bg', CONFIG.COLOR_BG);
</script>
</body>
</html>
```

- [ ] **Step 2: 브라우저에서 뼈대 확인**

`mcp__claude-in-chrome__navigate`로 `file:///C:/Users/jeodu/Desktop/VibeCoding/오내총(오늘은내가총무)/index.html` 열기 → `javascript_tool`로 아래 실행:

```javascript
JSON.stringify({
  title: document.title,
  configName: CONFIG.APP_SHORT,
  bg: getComputedStyle(document.body).backgroundColor,
})
```

Expected: `title`이 "오늘은 내가 총무", `configName`이 "오.내.총", `bg`가 크림톤(`rgb(255, 249, 242)`)으로 반환됨.

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "Stage 1: HTML 뼈대 및 CONFIG 추가"
```

---

### Task 2: store 추상화 + 도메인 함수 + 자체 테스트

**Files:**
- Modify: `index.html` — `<script>` 블록의 CONFIG 주입 코드 다음에 아래 섹션들을 순서대로 추가.

**Interfaces:**
- Consumes: `CONFIG.STORAGE_KEY`, `CONFIG.MEMBER_MIN`, `CONFIG.MEMBER_MAX`, `CONFIG.MEMBER_COLORS`, `CONFIG.MEMBER_EMOJIS`.
- Produces: `escapeHtml(str)`, `escapeAttr(str)`, `store.getAll()`, `store.get(id)`, `store.save(event)`, `store.remove(id)`, `createEvent(name, date)`, `addMember(event, name, emoji)`, `removeMember(event, memberId)`, `setOwner(event, memberId)`, `runSelfTests()`. 이후 태스크(3~5)는 이 함수들만 사용하고 localStorage를 직접 만지지 않는다.

- [ ] **Step 1: 유틸 함수 추가**

```javascript
  // ====== 유틸 ======
  function escapeHtml(str) {
    const div = document.createElement('div');
    div.textContent = String(str);
    return div.innerHTML;
  }

  function escapeAttr(str) {
    return String(str).replace(/"/g, '&quot;');
  }
```

- [ ] **Step 2: store 추상화 추가**

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

- [ ] **Step 3: 도메인 함수 추가 (모임/멤버 생성·검증)**

```javascript
  // ====== 도메인 함수: 모임/멤버 ======
  function createEvent(name, date) {
    return {
      id: crypto.randomUUID(),
      name,
      date,
      status: 'draft',
      account: { bank: '', number: '', holder: '' },
      members: [],
      items: [],
      createdAt: Date.now(),
      updatedAt: Date.now(),
    };
  }

  function nextMemberColor(members) {
    return CONFIG.MEMBER_COLORS[members.length % CONFIG.MEMBER_COLORS.length];
  }

  function addMember(event, name, emoji) {
    const trimmed = (name || '').trim();
    if (!trimmed) throw new Error('이름을 입력해주세요');
    if (event.members.some(m => m.name === trimmed)) {
      throw new Error('이미 존재하는 이름입니다');
    }
    if (event.members.length >= CONFIG.MEMBER_MAX) {
      throw new Error(`멤버는 최대 ${CONFIG.MEMBER_MAX}명까지 가능합니다`);
    }
    const member = {
      id: crypto.randomUUID(),
      name: trimmed,
      emoji: emoji || CONFIG.MEMBER_EMOJIS[event.members.length % CONFIG.MEMBER_EMOJIS.length],
      color: nextMemberColor(event.members),
      isOwner: event.members.length === 0,
    };
    event.members.push(member);
    return member;
  }

  function removeMember(event, memberId) {
    const idx = event.members.findIndex(m => m.id === memberId);
    if (idx === -1) return;
    const wasOwner = event.members[idx].isOwner;
    event.members.splice(idx, 1);
    if (wasOwner && event.members.length > 0) {
      event.members[0].isOwner = true;
    }
  }

  function setOwner(event, memberId) {
    event.members.forEach(m => { m.isOwner = (m.id === memberId); });
  }
```

- [ ] **Step 4: 자체 테스트 함수 추가**

```javascript
  // ====== 자체 테스트 (콘솔에서 runSelfTests() 실행) ======
  function runSelfTests() {
    const results = [];
    function assert(desc, cond) {
      results.push({ desc, pass: !!cond });
    }

    const ev = createEvent('테스트 모임', '2026-01-01');
    addMember(ev, '인우');
    addMember(ev, '찬우');
    addMember(ev, '정우');
    assert('첫 멤버가 총무로 지정됨', ev.members[0].isOwner === true);

    removeMember(ev, ev.members[0].id);
    assert('총무 삭제 시 다음 멤버로 이관', ev.members[0].isOwner === true);
    assert('총무 삭제 후 멤버 수 2명', ev.members.length === 2);

    let dupError = null;
    try { addMember(ev, '찬우'); } catch (e) { dupError = e; }
    assert('중복 이름 추가 시 에러', dupError !== null);

    let emptyError = null;
    try { addMember(ev, '   '); } catch (e) { emptyError = e; }
    assert('공백 이름 추가 시 에러', emptyError !== null);

    setOwner(ev, ev.members[1].id);
    assert('setOwner로 총무 변경', ev.members[1].isOwner === true && ev.members[0].isOwner === false);

    store.save(ev);
    const loaded = store.get(ev.id);
    assert('store 저장 후 조회 성공', !!loaded && loaded.id === ev.id);

    store.remove(ev.id);
    assert('store 삭제 후 조회 실패', store.get(ev.id) === null);

    console.table(results);
    const failed = results.filter(r => !r.pass);
    console.log(failed.length === 0 ? '✅ 전체 통과' : `❌ ${failed.length}건 실패`);
    return results;
  }
```

- [ ] **Step 5: 테스트 실행 및 확인**

Chrome에서 `index.html`을 새로고침 후 `javascript_tool`로 실행:

```javascript
JSON.stringify(runSelfTests())
```

Expected: 반환된 배열의 모든 원소가 `pass: true`. 하나라도 `false`면 Step 3의 해당 함수를 수정하고 재실행.

- [ ] **Step 6: 커밋**

```bash
git add index.html
git commit -m "Stage 1: store 추상화, 도메인 함수, 자체 테스트 추가"
```

---

### Task 3: 해시 라우터 + 공용 UI(토스트/다이얼로그/뱃지) + 홈 화면

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `store.getAll/get/save/remove`, `createEvent`, `escapeHtml`.
- Produces: `registerRoute(pattern, renderFn)`, `router()`, `showToast(message)`, `confirmDialog(message)` (Promise\<boolean\>), `renderBadge(text, variant)`, `renderHome()`. 라우트 `'/'`가 `renderHome`에 연결됨. `'/event/:id'` 패턴은 매칭 로직은 준비되지만 Task 4에서 등록되기 전까지는 홈으로 리다이렉트된다(핸들러 미등록 시 안전하게 폴백).

- [ ] **Step 1: 공용 UI 컴포넌트 CSS 추가**

`<style>` 블록 끝에 추가:

```css
  /* ====== 토스트 ====== */
  #toast-container {
    position: fixed;
    bottom: 24px;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    flex-direction: column;
    gap: 8px;
    z-index: 1000;
  }
  .toast {
    background: #3A2E28;
    color: #fff;
    padding: 10px 16px;
    border-radius: 10px;
    font-size: 14px;
    opacity: 0.95;
  }

  /* ====== 다이얼로그 ====== */
  .dialog-overlay {
    position: fixed;
    inset: 0;
    background: rgba(0,0,0,0.4);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
  }
  .dialog-box {
    background: #fff;
    border-radius: 16px;
    padding: 20px;
    width: min(320px, 90vw);
    display: flex;
    flex-direction: column;
    gap: 16px;
  }
  .dialog-actions {
    display: flex;
    gap: 8px;
    justify-content: flex-end;
  }

  /* ====== 뱃지 ====== */
  .badge {
    display: inline-block;
    padding: 3px 10px;
    border-radius: 999px;
    font-size: 12px;
    font-weight: 600;
  }
  .badge-default { background: #EFE6DC; color: #6B5B4B; }
  .badge-primary { background: var(--color-primary); color: #fff; }
  .badge-success { background: #4CAF50; color: #fff; }

  /* ====== 홈: 모임 카드 ====== */
  .event-list { display: flex; flex-direction: column; gap: 12px; }
  .event-card {
    background: #fff;
    border-radius: 16px;
    padding: 16px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.06);
    display: flex;
    flex-direction: column;
    gap: 6px;
  }
  .card-top { display: flex; justify-content: space-between; align-items: center; }
  .event-name { font-size: 16px; font-weight: 700; }
  .card-menu-btn {
    background: none; border: none; font-size: 18px; color: #9A8C7C;
    min-width: 44px; min-height: 44px;
  }
  .card-meta { display: flex; gap: 8px; align-items: center; color: #6B5B4B; font-size: 13px; }
  .card-members { font-size: 18px; }
  .card-total { font-size: 18px; font-weight: 700; color: var(--color-primary); }

  .empty-state {
    text-align: center;
    padding: 64px 16px;
    color: #9A8C7C;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }
  .empty-emoji { font-size: 40px; }

  .fab {
    position: fixed;
    bottom: 24px;
    left: 50%;
    transform: translateX(-50%);
    background: var(--color-primary);
    color: #fff;
    border: none;
    border-radius: 999px;
    padding: 14px 24px;
    font-size: 15px;
    font-weight: 700;
    box-shadow: 0 4px 12px rgba(0,0,0,0.15);
    min-height: 44px;
  }
```

- [ ] **Step 2: 공용 UI 컴포넌트 JS 추가**

```javascript
  // ====== 공용 UI 컴포넌트 ======
  function showToast(message) {
    let container = document.getElementById('toast-container');
    if (!container) {
      container = document.createElement('div');
      container.id = 'toast-container';
      document.body.appendChild(container);
    }
    const toast = document.createElement('div');
    toast.className = 'toast';
    toast.textContent = message;
    container.appendChild(toast);
    setTimeout(() => toast.remove(), 2000);
  }

  function confirmDialog(message) {
    return new Promise((resolve) => {
      const overlay = document.createElement('div');
      overlay.className = 'dialog-overlay';
      overlay.innerHTML = `
        <div class="dialog-box">
          <p class="dialog-message"></p>
          <div class="dialog-actions">
            <button type="button" class="btn btn-ghost" data-action="cancel">취소</button>
            <button type="button" class="btn btn-danger" data-action="confirm">확인</button>
          </div>
        </div>
      `;
      overlay.querySelector('.dialog-message').textContent = message;
      overlay.addEventListener('click', (e) => {
        const action = e.target.dataset.action;
        if (action === 'confirm') { overlay.remove(); resolve(true); }
        else if (action === 'cancel' || e.target === overlay) { overlay.remove(); resolve(false); }
      });
      document.body.appendChild(overlay);
    });
  }

  function renderBadge(text, variant) {
    return `<span class="badge badge-${variant || 'default'}">${escapeHtml(text)}</span>`;
  }
```

- [ ] **Step 3: 해시 라우터 추가**

```javascript
  // ====== 라우터 ======
  const routes = {};

  function registerRoute(pattern, renderFn) {
    routes[pattern] = renderFn;
  }

  function parseHash() {
    return location.hash.replace(/^#/, '') || '/';
  }

  function matchRoute(hash) {
    if (hash === '/' && routes['/']) {
      return { render: routes['/'], params: {} };
    }
    const eventMatch = hash.match(/^\/event\/([^/]+)$/);
    if (eventMatch && routes['/event/:id']) {
      return { render: routes['/event/:id'], params: { id: eventMatch[1] } };
    }
    return null;
  }

  function router() {
    const hash = parseHash();
    const matched = matchRoute(hash);
    if (!matched) {
      if (hash !== '/') { location.hash = '#/'; }
      else if (routes['/']) { routes['/'](); }
      return;
    }
    matched.render(matched.params);
  }

  window.addEventListener('hashchange', router);
```

- [ ] **Step 4: 홈 화면 추가**

```javascript
  // ====== 화면: 홈 (모임 목록) ======
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

  function renderEmptyState() {
    return `
      <div class="empty-state">
        <div class="empty-emoji">🦊🧾</div>
        <p>첫 모임을 만들어보세요</p>
      </div>
    `;
  }

  function renderEventCard(ev) {
    const emojis = ev.members.map(m => m.emoji).join(' ');
    return `
      <div class="event-card" data-id="${ev.id}">
        <div class="card-top">
          <span class="event-name">${escapeHtml(ev.name)}</span>
          <button type="button" class="card-menu-btn" data-id="${ev.id}">⋯</button>
        </div>
        <div class="card-meta">
          <span class="event-date">${escapeHtml(ev.date)}</span>
          ${renderBadge(ev.status === 'settled' ? '정산 완료' : '작성 중', ev.status === 'settled' ? 'success' : 'default')}
        </div>
        <div class="card-members">${emojis}</div>
        <div class="card-total">₩0</div>
      </div>
    `;
  }

  function onCreateEvent() {
    const today = new Date().toISOString().slice(0, 10);
    const ev = createEvent('새 모임', today);
    store.save(ev);
    location.hash = `#/event/${ev.id}`;
  }

  async function onDeleteEvent(id) {
    const ev = store.get(id);
    if (!ev) return;
    const ok = await confirmDialog(`"${ev.name}" 모임을 삭제할까요?`);
    if (ok) {
      store.remove(id);
      renderHome();
    }
  }

  // ====== 라우트 등록 & 초기 실행 ======
  registerRoute('/', renderHome);
  window.addEventListener('DOMContentLoaded', router);
```

- [ ] **Step 5: 브라우저 검증**

Chrome에서 `index.html` 새로고침 후:
1. 빈 상태("첫 모임을 만들어보세요")가 보이는지 `javascript_tool`로 `document.querySelector('.empty-state')?.textContent` 확인 (Expected: null이 아니고 "첫 모임을 만들어보세요" 포함).
2. `computer` 도구로 "+ 새 모임" 버튼 클릭 → 새 모임이 store에 생성되고 `location.hash`가 `#/event/{id}`로 바뀌지만 아직 라우트가 없으므로 홈으로 리다이렉트되어 카드가 1개 보이는지 확인 (`document.querySelectorAll('.event-card').length === 1`).
3. `#/nope`으로 이동 후 `location.hash`가 `#/`로 바뀌는지 확인.

- [ ] **Step 6: 커밋**

```bash
git add index.html
git commit -m "Stage 1: 해시 라우터, 공용 UI, 홈 화면 추가"
```

---

### Task 4: 모임 설정 화면 — 기본 정보 입력 + 자동 저장

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `store.get/save`, `escapeHtml`, `escapeAttr`, `registerRoute`.
- Produces: `renderEventSettings(params)` (라우트 `'/event/:id'`에 등록), `scheduleSave(ev)`. Task 5는 같은 `renderEventSettings` 함수의 멤버 섹션을 확장한다.

- [ ] **Step 1: 폼/저장 인디케이터 CSS 추가**

```css
  /* ====== 모임 설정: 폼 ====== */
  .btn-back {
    background: none; border: none; font-size: 20px;
    min-width: 44px; min-height: 44px;
  }
  .save-indicator {
    font-size: 13px;
    color: #4CAF50;
    min-width: 64px;
    text-align: right;
  }
  .field-label {
    font-size: 13px;
    font-weight: 600;
    color: #6B5B4B;
  }
  .input {
    border: 1px solid #E0D6CA;
    border-radius: 10px;
    padding: 10px 12px;
    font-size: 15px;
    min-height: 44px;
    width: 100%;
  }
  .account-fields {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }
  .field-error {
    color: #E74C3C;
    font-size: 13px;
    min-height: 18px;
  }
  .field-hint {
    color: #9A8C7C;
    font-size: 13px;
  }
```

- [ ] **Step 2: `renderEventSettings` (모임 정보 부분만) 추가**

```javascript
  // ====== 화면: 모임 설정 ======
  let saveDebounceTimer = null;

  function renderEventSettings(params) {
    const app = document.getElementById('app');
    const ev = store.get(params.id);
    if (!ev) { location.hash = '#/'; return; }

    app.innerHTML = `
      <header class="topbar">
        <button type="button" class="btn-back" id="btn-back">←</button>
        <h1>모임 설정</h1>
        <span id="save-indicator" class="save-indicator"></span>
      </header>
      <main class="screen">
        <section class="card-section">
          <label class="field-label" for="input-name">모임명</label>
          <input id="input-name" class="input" type="text" value="${escapeAttr(ev.name)}" placeholder="모임명을 입력하세요" />
          <label class="field-label" for="input-date">날짜</label>
          <input id="input-date" class="input" type="date" value="${escapeAttr(ev.date)}" />
          <label class="field-label">계좌 정보 (선택)</label>
          <div class="account-fields">
            <input id="input-bank" class="input" type="text" placeholder="은행" value="${escapeAttr(ev.account.bank)}" />
            <input id="input-number" class="input" type="text" placeholder="계좌번호" value="${escapeAttr(ev.account.number)}" />
            <input id="input-holder" class="input" type="text" placeholder="예금주" value="${escapeAttr(ev.account.holder)}" />
          </div>
        </section>

        <button type="button" id="btn-goto-items" class="btn btn-disabled" disabled>결제 입력으로 →</button>
      </main>
    `;

    document.getElementById('btn-back').addEventListener('click', () => { location.hash = '#/'; });

    ['input-name', 'input-date', 'input-bank', 'input-number', 'input-holder'].forEach(id => {
      document.getElementById(id).addEventListener('input', () => scheduleSave(ev));
    });

    document.getElementById('btn-goto-items').addEventListener('click', () => {
      showToast('Stage 2에서 열립니다');
    });
  }

  function scheduleSave(ev) {
    clearTimeout(saveDebounceTimer);
    const indicator = document.getElementById('save-indicator');
    if (indicator) indicator.textContent = '';
    saveDebounceTimer = setTimeout(() => {
      const nameValue = document.getElementById('input-name').value.trim();
      ev.name = nameValue || ev.name;
      ev.date = document.getElementById('input-date').value || ev.date;
      ev.account.bank = document.getElementById('input-bank').value;
      ev.account.number = document.getElementById('input-number').value;
      ev.account.holder = document.getElementById('input-holder').value;
      store.save(ev);
      const el = document.getElementById('save-indicator');
      if (el) el.textContent = '저장됨 ✓';
    }, 500);
  }

  registerRoute('/event/:id', renderEventSettings);
```

`registerRoute('/event/:id', renderEventSettings);` 는 Task 3의 `registerRoute('/', renderHome);` 바로 아래, `DOMContentLoaded` 리스너 등록 이전에 추가한다.

- [ ] **Step 3: 브라우저 검증**

Chrome에서 새로고침 후 `javascript_tool`로 실행:

```javascript
const ev = createEvent('여우회 강화도 나들이', '2026-07-12');
store.save(ev);
location.hash = '#/event/' + ev.id;
```

이어서 확인:
1. `document.getElementById('input-name').value === '여우회 강화도 나들이'`.
2. `computer` 도구로 모임명 입력을 "여우회 강화도 1차"로 수정 → 600ms 대기 → `document.getElementById('save-indicator').textContent === '저장됨 ✓'`.
3. 새로고침 후 같은 해시로 재진입 시 `store.get(ev.id).name === '여우회 강화도 1차'` (자동 저장 확인).
4. "결제 입력으로 →" 버튼 클릭 시 토스트 "Stage 2에서 열립니다" 노출 확인.

- [ ] **Step 4: 커밋**

```bash
git add index.html
git commit -m "Stage 1: 모임 설정 화면(기본 정보) 및 자동 저장 추가"
```

---

### Task 5: 모임 설정 화면 — 멤버 관리

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `addMember`, `removeMember`, `setOwner`, `store.save`, `escapeHtml`, `renderBadge`, `renderEventSettings(params)`(재렌더용).
- Produces: `renderMemberChip(m)`, `bindMemberListEvents(ev, params)`. Task 4의 `renderEventSettings`를 확장한다.

- [ ] **Step 1: 멤버 UI CSS 추가**

```css
  /* ====== 모임 설정: 멤버 ====== */
  .member-list {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }
  .member-chip {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 8px 10px;
    border-radius: 12px;
  }
  .member-owner-radio input { width: 18px; height: 18px; }
  .member-emoji { font-size: 18px; }
  .member-name { flex: 1; font-weight: 600; }
  .member-remove-btn {
    background: none; border: none; color: #6B5B4B; font-size: 14px;
    min-width: 32px; min-height: 32px;
  }
  .member-add-form {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }
  .emoji-picker {
    display: grid;
    grid-template-columns: repeat(6, 1fr);
    gap: 6px;
  }
  .emoji-btn {
    background: #F5EEE4;
    border: 2px solid transparent;
    border-radius: 10px;
    font-size: 20px;
    min-height: 44px;
  }
  .emoji-btn.selected {
    border-color: var(--color-primary);
    background: #fff;
  }
```

- [ ] **Step 2: `renderEventSettings`에 멤버 섹션 삽입**

Task 4의 `renderEventSettings` 안에서 `</section>` (계좌 정보 섹션 닫는 태그) 바로 다음, `<button type="button" id="btn-goto-items"` 앞에 아래 블록을 삽입:

```javascript
        <section class="card-section">
          <h2>멤버 (${ev.members.length}/${CONFIG.MEMBER_MAX})</h2>
          <div id="member-list" class="member-list">${ev.members.map(renderMemberChip).join('')}</div>
          ${ev.members.length < CONFIG.MEMBER_MIN ? `<div class="field-hint">최소 ${CONFIG.MEMBER_MIN}명 이상 등록해주세요</div>` : ''}
          <div id="member-error" class="field-error"></div>
          <div class="member-add-form">
            <input id="input-member-name" class="input" type="text" placeholder="이름" />
            <div id="emoji-picker" class="emoji-picker">
              ${CONFIG.MEMBER_EMOJIS.map((e, i) => `<button type="button" class="emoji-btn${i === 0 ? ' selected' : ''}" data-emoji="${e}">${e}</button>`).join('')}
            </div>
            <button type="button" id="btn-add-member" class="btn btn-primary">멤버 추가</button>
          </div>
        </section>
```

그리고 `renderEventSettings` 함수 본문의 이벤트 바인딩 코드(현재 `document.getElementById('btn-goto-items')...` 아래) 끝에 추가:

```javascript
    let selectedEmoji = CONFIG.MEMBER_EMOJIS[0];

    document.querySelectorAll('.emoji-btn').forEach(btn => {
      btn.addEventListener('click', () => {
        document.querySelectorAll('.emoji-btn').forEach(b => b.classList.remove('selected'));
        btn.classList.add('selected');
        selectedEmoji = btn.dataset.emoji;
      });
    });

    document.getElementById('btn-add-member').addEventListener('click', () => {
      const nameInput = document.getElementById('input-member-name');
      const errorEl = document.getElementById('member-error');
      errorEl.textContent = '';
      try {
        addMember(ev, nameInput.value, selectedEmoji);
        store.save(ev);
        renderEventSettings(params);
      } catch (err) {
        errorEl.textContent = err.message;
      }
    });

    bindMemberListEvents(ev, params);
```

- [ ] **Step 3: `renderMemberChip` / `bindMemberListEvents` 추가**

`renderEventSettings` 함수 바로 아래(전역 스코프)에 추가:

```javascript
  function renderMemberChip(m) {
    return `
      <div class="member-chip" style="background:${m.color}">
        <label class="member-owner-radio">
          <input type="radio" name="owner" value="${m.id}" ${m.isOwner ? 'checked' : ''} />
        </label>
        <span class="member-emoji">${m.emoji}</span>
        <span class="member-name">${escapeHtml(m.name)}</span>
        ${m.isOwner ? renderBadge('총무', 'primary') : ''}
        <button type="button" class="member-remove-btn" data-id="${m.id}">✕</button>
      </div>
    `;
  }

  function bindMemberListEvents(ev, params) {
    document.querySelectorAll('input[name="owner"]').forEach(radio => {
      radio.addEventListener('change', () => {
        setOwner(ev, radio.value);
        store.save(ev);
        renderEventSettings(params);
      });
    });
    document.querySelectorAll('.member-remove-btn').forEach(btn => {
      btn.addEventListener('click', () => {
        removeMember(ev, btn.dataset.id);
        store.save(ev);
        renderEventSettings(params);
      });
    });
  }
```

- [ ] **Step 4: 브라우저 검증**

Chrome에서 새로고침 후 Task 4에서 만든 이벤트로 이동해 `computer`/`javascript_tool`로:
1. 멤버 5인 추가: 인우·찬우·정표·정우·은찬 (이모지는 기본 순환 배정). 각 추가 후 `document.querySelectorAll('.member-chip').length` 증가 확인.
2. 이름을 "찬우"로 다시 추가 시도 → `#member-error` 텍스트에 "이미 존재하는 이름입니다" 표시 확인.
3. 이름을 공백("   ")으로 추가 시도 → `#member-error`에 "이름을 입력해주세요" 표시 확인.
4. "정우" 라디오를 총무로 선택 → `store.get(ev.id).members.find(m => m.name === '정우').isOwner === true` 확인.
5. 총무(정우)의 ✕ 버튼 클릭 → `store.get(ev.id).members[0].isOwner === true` (자동 이관) 확인.

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "Stage 1: 모임 설정 화면 멤버 관리 기능 추가"
```

---

### Task 6: 반응형 마감 + 완료 기준 전체 검증 + 문서 업데이트

**Files:**
- Modify: `index.html`
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: 전체 앱.
- Produces: 없음 (마감 태스크).

- [ ] **Step 1: 반응형 미디어쿼리 보강**

`<style>` 블록 끝에 추가:

```css
  /* ====== 반응형 보강 ====== */
  @media (min-width: 561px) {
    #app { padding-left: 0; padding-right: 0; }
  }
  @media (max-width: 400px) {
    .card-section { padding: 12px; }
    .emoji-picker { grid-template-columns: repeat(5, 1fr); }
  }
```

- [ ] **Step 2: 375px~데스크톱 레이아웃 확인**

Chrome에서 `resize_window`(또는 뷰포트 크기 조정)로 375px, 390px, 768px, 1280px 각각에서 홈 화면과 모임 설정 화면을 캡처해 가로 스크롤/요소 겹침이 없는지 확인. 특히 `.emoji-picker`와 `.event-card` 내부 요소가 폭을 벗어나지 않는지 `document.body.scrollWidth <= window.innerWidth` 로 확인.

- [ ] **Step 3: 완료 기준 6개 항목 전체 재검증**

`docs/superpowers/specs/2026-07-12-stage1-skeleton-design.md` §9의 6개 항목을 처음부터 순서대로 재실행:
1. "여우회 강화도 나들이" 모임 생성 → 멤버 5인(총무=정우) 등록 → 홈에서 카드 확인.
2. 새로고침 후 데이터 유지 확인.
3. 멤버 이름 중복 시 차단 메시지 확인.
4. 총무 삭제 시 자동 이관 확인.
5. `#/nope` 접근 시 홈으로 리다이렉트 확인.
6. 375px~1280px 레이아웃 확인(Step 2에서 확인 완료).

모두 통과해야 다음 단계로 진행 가능. 실패 항목이 있으면 해당 태스크로 돌아가 수정 후 재검증.

- [ ] **Step 4: `CLAUDE.md`에 Stage 1 완료 상태 반영**

`CLAUDE.md`의 "Project status" 섹션에서 "No code exists yet" 문장을 아래로 교체:

```markdown
## Project status

Stage 1 (skeleton: hash routing, CONFIG, event/member CRUD) is implemented in `index.html`. Stages 2-5 (settlement engine, receipts, Firebase sync, PWA polish) are not yet implemented — see `오내총_PRD_v1.md` §9 and `docs/superpowers/specs/` for design docs, and `docs/superpowers/plans/` for implementation plans.
```

- [ ] **Step 5: 최종 커밋**

```bash
git add index.html CLAUDE.md
git commit -m "Stage 1: 반응형 마감 및 완료 기준 검증"
```
