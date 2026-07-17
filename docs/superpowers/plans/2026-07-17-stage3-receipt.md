# Stage 3 Implementation Plan: 감성 영수증 — 디자인 · 사진 첨부 · 이미지 저장/공유

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-member "감성 영수증" (thermal-receipt-style) generation to `index.html`: a dedicated route that renders a real DOM receipt from `calcSettlement` data, lets the 총무 attach/filter a highlight photo (stored only in `localStorage`), and captures the receipt to a shareable/downloadable PNG via `html2canvas`.

**Architecture:** Single-file addition, same style as Stage 1/2. A new `photoStore` object (parallel to `store`) owns all photo persistence. Pure canvas functions handle image resize/filtering, decoupled from the DOM. `renderReceiptView`/`renderReceiptDom` render the receipt as real markup (not canvas-drawn) so `html2canvas` can capture it. Two dynamically-created overlay modals (photo attach, photo manager) reuse the existing `.dialog-overlay` interaction pattern from `confirmDialog`. Router extended with one more hardcoded-regex route, same style as Stage 2.

**Tech Stack:** Vanilla JS/HTML/CSS (no change). New CDN additions: Galmuri webfont, html2canvas 1.4.1.

## Global Constraints

- Single `index.html` file. No build tools, no npm, no bundler, no framework.
- CDN-only new dependencies: Galmuri webfont CSS, `html2canvas@1.4.1`.
- Comments in Korean.
- `CONFIG` is the single source of truth for tunable constants — new literal CSS colors introduced for the receipt design (e.g. `#2B2420`, `#D64545`) are NOT added to `CONFIG`, following the Stage 1/2 precedent that only the 3 core theme colors live there.
- Photos live ONLY in `localStorage` (`CONFIG.PHOTOS_KEY`), never touch `event`/Firestore-bound data. `event.*` schema is not modified in this stage.
- Money is never recomputed independently in the receipt — always sourced from a single `calcSettlement(event)` call per render, per CLAUDE.md's "settlement is never persisted, always recomputed" invariant.
- Every user-controlled string (item name/emoji, member name, account fields) goes through `escapeHtml`/`escapeAttr` before entering `innerHTML`.
- No CSS `filter:` for photo processing — filters are baked into pixel data via `<canvas>` at save time (html2canvas does not reliably support CSS filters).
- `document.fonts.ready` must be awaited before every `html2canvas` capture.
- Elements that must not appear in the captured PNG (copy buttons etc.) carry `data-no-capture` and are excluded via html2canvas's `ignoreElements`.
- Chrome browser automation was unavailable for the entirety of Stage 2's work in this session. Every task below gives a Node+jsdom fallback verification method for logic/DOM tasks, and explicitly flags where only a real browser can confirm correctness (canvas pixel output, html2canvas rasterization, file download/share).

---

### Task 1: CONFIG 확장 + `photoStore` 데이터 계층 + `hashString` + 회귀 테스트

**Files:**
- Modify: `index.html` (`<script>` 블록: CONFIG, 유틸 섹션, STORE 섹션, `runRegressionTests`)

**Interfaces:**
- Consumes: `CONFIG` (기존), `runRegressionTests`/`assert` (기존, Stage 2)
- Produces: `photoStore.{get,set,remove,removeByEvent,listByEvent,usage}`, `hashString(str) → number`, `CONFIG.{PHOTOS_KEY,RECEIPT_WIDTH,RECEIPT_SCALE,PHOTO_MAX_EDGE,PHOTO_QUALITY,RECEIPT_FONT_URL,RECEIPT_MENTS,PHOTO_NOTICE_KEY}`

- [ ] **Step 1: CONFIG 필드 추가**

Find:
```javascript
    CURRENCY_LOCALE: 'ko-KR',
    DEV_MODE: false,
  };
```

Replace with:
```javascript
    CURRENCY_LOCALE: 'ko-KR',
    DEV_MODE: false,
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
  };
```

- [ ] **Step 2: Galmuri 웹폰트 동적 로드**

Find:
```javascript
  // CONFIG 색상값을 CSS 커스텀 프로퍼티로 주입 (CSS에 색상 하드코딩 금지)
  document.documentElement.style.setProperty('--color-primary', CONFIG.COLOR_PRIMARY);
  document.documentElement.style.setProperty('--color-bg', CONFIG.COLOR_BG);
```

Replace with:
```javascript
  // CONFIG 색상값을 CSS 커스텀 프로퍼티로 주입 (CSS에 색상 하드코딩 금지)
  document.documentElement.style.setProperty('--color-primary', CONFIG.COLOR_PRIMARY);
  document.documentElement.style.setProperty('--color-bg', CONFIG.COLOR_BG);

  // 영수증 도트 폰트 동적 로드 (CONFIG.RECEIPT_FONT_URL을 단일 출처로 유지)
  const receiptFontLink = document.createElement('link');
  receiptFontLink.rel = 'stylesheet';
  receiptFontLink.href = CONFIG.RECEIPT_FONT_URL;
  document.head.appendChild(receiptFontLink);
```

- [ ] **Step 3: `hashString` 유틸 추가**

Find:
```javascript
  function escapeAttr(str) {
    return String(str).replace(/"/g, '&quot;');
  }
```

Replace with:
```javascript
  function escapeAttr(str) {
    return String(str).replace(/"/g, '&quot;');
  }

  // 결정론적 해시 (FNV-1a 32bit 변형) — 영수증 바코드 모양·랜덤 멘트 선택 시드로 사용
  function hashString(str) {
    let hash = 0x811c9dc5;
    for (let i = 0; i < str.length; i++) {
      hash ^= str.charCodeAt(i);
      hash = Math.imul(hash, 0x01000193);
    }
    return hash >>> 0;
  }
```

- [ ] **Step 4: `photoStore` 추가**

Find:
```javascript
    remove(id) {
      const events = this._readAll().filter(e => e.id !== id);
      this._writeAll(events);
    },
  };

  // ====== 도메인 함수: 모임/멤버 ======
```

Replace with:
```javascript
    remove(id) {
      const events = this._readAll().filter(e => e.id !== id);
      this._writeAll(events);
    },
  };

  // ====== PHOTO STORE (사진 전용 localStorage 추상화, event 데이터와 분리) ======
  const photoStore = {
    _readAll() {
      const raw = localStorage.getItem(CONFIG.PHOTOS_KEY);
      return raw ? JSON.parse(raw) : {};
    },
    _writeAll(map) {
      localStorage.setItem(CONFIG.PHOTOS_KEY, JSON.stringify(map));
    },
    _key(eventId, memberId) {
      return `${eventId}:${memberId}`;
    },
    get(eventId, memberId) {
      return this._readAll()[this._key(eventId, memberId)] || null;
    },
    set(eventId, memberId, dataUrl) {
      const map = this._readAll();
      map[this._key(eventId, memberId)] = dataUrl;
      this._writeAll(map); // QuotaExceededError는 호출부에서 처리
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
      return Object.keys(map)
        .filter(k => k.startsWith(`${eventId}:`))
        .map(k => ({ key: k, memberId: k.split(':')[1], dataUrl: map[k] }));
    },
    usage() {
      const raw = localStorage.getItem(CONFIG.PHOTOS_KEY) || '';
      return Math.round(raw.length / 1024); // KB 근사치
    },
  };

  // ====== 도메인 함수: 모임/멤버 ======
```

- [ ] **Step 5: 회귀 테스트에 `hashString`/`photoStore` 검증 추가**

Find:
```javascript
    console.table(results);
    const failed = results.filter(r => !r.pass);
    console.log(failed.length === 0 ? '✅ 회귀 테스트 전체 통과' : `❌ ${failed.length}건 실패`);
    return results;
  }
```

Replace with:
```javascript
    // ---- hashString: 결정론적 해시 ----
    assert('hashString: 같은 입력 → 같은 값', hashString('오내총') === hashString('오내총'));
    assert('hashString: 다른 입력 → 다른 값', hashString('오내총') !== hashString('오내총2'));
    assert('hashString: 32비트 부호없는 정수 반환', Number.isInteger(hashString('test')) && hashString('test') >= 0);

    // ---- photoStore CRUD ----
    const PHOTO_TEST_EVENT = '__test_event__';
    const testDataUrl = 'data:image/jpeg;base64,AAAA';
    photoStore.set(PHOTO_TEST_EVENT, 'm1', testDataUrl);
    assert('photoStore: set 후 get 동일값', photoStore.get(PHOTO_TEST_EVENT, 'm1') === testDataUrl);
    assert('photoStore: 없는 키는 null', photoStore.get(PHOTO_TEST_EVENT, 'm2') === null);
    photoStore.set(PHOTO_TEST_EVENT, 'm2', testDataUrl);
    assert('photoStore: listByEvent 2건', photoStore.listByEvent(PHOTO_TEST_EVENT).length === 2);
    photoStore.remove(PHOTO_TEST_EVENT, 'm1');
    assert('photoStore: remove 후 get null', photoStore.get(PHOTO_TEST_EVENT, 'm1') === null);
    assert('photoStore: remove 후 listByEvent 1건', photoStore.listByEvent(PHOTO_TEST_EVENT).length === 1);
    photoStore.removeByEvent(PHOTO_TEST_EVENT);
    assert('photoStore: removeByEvent 후 listByEvent 0건', photoStore.listByEvent(PHOTO_TEST_EVENT).length === 0);
    assert('photoStore: usage()는 숫자 반환', typeof photoStore.usage() === 'number');

    console.table(results);
    const failed = results.filter(r => !r.pass);
    console.log(failed.length === 0 ? '✅ 회귀 테스트 전체 통과' : `❌ ${failed.length}건 실패`);
    return results;
  }
```

- [ ] **Step 6: 검증**

Chrome 브라우저 자동화가 가능하면: `http://localhost:8123/index.html` 콘솔에서 `CONFIG.DEV_MODE = true`로 바꾸고 새로고침하거나, 콘솔에서 직접 `runRegressionTests()` 실행 → `✅ 회귀 테스트 전체 통과` 확인 (총 48개 assertion: 기존 39 + 이번 9개).

Chrome 자동화가 불가능하면(이번 세션 Stage 2 작업 시 계속 연결 실패했음): Node + jsdom으로 실제 `index.html`을 그대로 로드해 검증한다 (프로젝트에 jsdom을 devDependency로 추가하지 않음 — 별도 임시 디렉터리에서 실행):

```bash
mkdir -p /tmp/onct-verify && cd /tmp/onct-verify
npm init -y && npm install jsdom --silent
```

```javascript
// /tmp/onct-verify/run.js
const fs = require('fs');
const { JSDOM, VirtualConsole } = require('jsdom');
let html = fs.readFileSync('<repo>/index.html', 'utf8').replace('DEV_MODE: false', 'DEV_MODE: true');
const logs = [];
const vc = new VirtualConsole();
['log', 'warn', 'error'].forEach(level => vc.on(level, (...a) => logs.push([level, ...a])));
new JSDOM(html, { url: 'http://localhost:8123/index.html', runScripts: 'dangerously', resources: 'usable', pretendToBeVisual: true, virtualConsole: vc });
setTimeout(() => { console.log(logs.map(l => l.join(' ')).join('\n')); }, 800);
```

```bash
node run.js
```

Expected output includes `✅ 회귀 테스트 전체 통과`. `<repo>`는 실제 프로젝트 절대경로로 치환.

- [ ] **Step 7: 커밋**

```bash
git add index.html
git commit -m "Stage 3: CONFIG 확장 + photoStore 데이터 계층 + hashString + 회귀 테스트"
```

---

### Task 2: 사진 압축·필터 파이프라인

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: 없음 (순수 `<canvas>`/`FileReader`/`Image` API만 사용)
- Produces: `resizeImageToCanvas(file, maxEdge) → Promise<HTMLCanvasElement>`, `applyPhotoFilter(sourceCanvas, filterName) → HTMLCanvasElement` (`filterName`: `'original'|'grayscale'|'dither'`), `canvasToCompressedDataURL(canvas, quality) → string`

- [ ] **Step 1: 파이프라인 함수 추가**

Find (Task 1이 추가한 `photoStore`의 마지막 부분):
```javascript
    usage() {
      const raw = localStorage.getItem(CONFIG.PHOTOS_KEY) || '';
      return Math.round(raw.length / 1024); // KB 근사치
    },
  };

  // ====== 도메인 함수: 모임/멤버 ======
```

Replace with:
```javascript
    usage() {
      const raw = localStorage.getItem(CONFIG.PHOTOS_KEY) || '';
      return Math.round(raw.length / 1024); // KB 근사치
    },
  };

  // ====== 사진 압축·필터 파이프라인 (순수 canvas 처리, DOM 부수효과 없음) ======
  function resizeImageToCanvas(file, maxEdge) {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onerror = () => reject(new Error('이미지를 읽을 수 없습니다'));
      reader.onload = () => {
        const img = new Image();
        img.onerror = () => reject(new Error('이미지를 불러올 수 없습니다'));
        img.onload = () => {
          const ratio = Math.min(1, maxEdge / Math.max(img.width, img.height));
          const w = Math.round(img.width * ratio);
          const h = Math.round(img.height * ratio);
          const canvas = document.createElement('canvas');
          canvas.width = w;
          canvas.height = h;
          canvas.getContext('2d').drawImage(img, 0, 0, w, h);
          resolve(canvas);
        };
        img.src = reader.result;
      };
      reader.readAsDataURL(file);
    });
  }

  function cloneCanvas(source) {
    const canvas = document.createElement('canvas');
    canvas.width = source.width;
    canvas.height = source.height;
    canvas.getContext('2d').drawImage(source, 0, 0);
    return canvas;
  }

  function applyGrayscaleFilter(sourceCanvas) {
    const canvas = cloneCanvas(sourceCanvas);
    const ctx = canvas.getContext('2d');
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    const data = imageData.data;
    for (let i = 0; i < data.length; i += 4) {
      const lum = 0.299 * data[i] + 0.587 * data[i + 1] + 0.114 * data[i + 2];
      // 감열지 감성을 위해 대비를 살짝 올림 (128 기준 1.15배 확장)
      const contrasted = Math.min(255, Math.max(0, (lum - 128) * 1.15 + 128));
      data[i] = data[i + 1] = data[i + 2] = contrasted;
    }
    ctx.putImageData(imageData, 0, 0);
    return canvas;
  }

  const BAYER_4X4 = [
    [0, 8, 2, 10],
    [12, 4, 14, 6],
    [3, 11, 1, 9],
    [15, 7, 13, 5],
  ];

  function applyDitherFilter(sourceCanvas) {
    const gray = applyGrayscaleFilter(sourceCanvas);
    const canvas = cloneCanvas(gray);
    const ctx = canvas.getContext('2d');
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    const data = imageData.data;
    for (let y = 0; y < canvas.height; y++) {
      for (let x = 0; x < canvas.width; x++) {
        const idx = (y * canvas.width + x) * 4;
        const threshold = (BAYER_4X4[y % 4][x % 4] / 16) * 255;
        const value = data[idx] > threshold ? 255 : 0;
        data[idx] = data[idx + 1] = data[idx + 2] = value;
      }
    }
    ctx.putImageData(imageData, 0, 0);
    return canvas;
  }

  function applyPhotoFilter(sourceCanvas, filterName) {
    if (filterName === 'grayscale') return applyGrayscaleFilter(sourceCanvas);
    if (filterName === 'dither') return applyDitherFilter(sourceCanvas);
    return cloneCanvas(sourceCanvas);
  }

  function canvasToCompressedDataURL(canvas, quality) {
    return canvas.toDataURL('image/jpeg', quality);
  }

  // ====== 도메인 함수: 모임/멤버 ======
```

- [ ] **Step 2: 검증 (반드시 실제 브라우저 — jsdom은 `<canvas>` 2D 컨텍스트를 지원하지 않아 이 태스크는 자동 검증 불가)**

`http://localhost:8123/index.html` 콘솔에서:

```javascript
const c = document.createElement('canvas');
c.width = 100; c.height = 60;
const ctx = c.getContext('2d');
ctx.fillStyle = 'red'; ctx.fillRect(0, 0, 50, 60);
ctx.fillStyle = 'blue'; ctx.fillRect(50, 0, 50, 60);

const gray = applyPhotoFilter(c, 'grayscale');
const dither = applyPhotoFilter(c, 'dither');
const original = applyPhotoFilter(c, 'original');
console.log('크기 유지:', gray.width === 100 && gray.height === 60);
console.log('grayscale 픽셀 R=G=B:', (() => { const d = gray.getContext('2d').getImageData(10,10,1,1).data; return d[0]===d[1] && d[1]===d[2]; })());
console.log('dither는 0 또는 255만:', (() => { const d = dither.getContext('2d').getImageData(0,0,100,60).data; for (let i=0;i<d.length;i+=4) { if (d[i]!==0 && d[i]!==255) return false; } return true; })());
console.log('original 색상 보존:', (() => { const d = original.getContext('2d').getImageData(10,10,1,1).data; return d[0] > 200 && d[2] < 50; })()); // 왼쪽은 빨강
```

모두 `true`가 출력되어야 한다. 이 브라우저 콘솔 확인이 불가능하면(Chrome 자동화 미연결) 이 태스크는 "미검증"으로 최종 보고서에 명시하고, Task 7의 완료 기준 검증 단계에서 다시 시도한다.

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "Stage 3: 사진 압축·필터(원본/흑백/도트 Bayer dithering) 파이프라인"
```

---

### Task 3: 라우터 확장 + 영수증 화면 전체 (탭·DOM 8섹션·CSS)

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `calcSettlement`, `photoStore.get`, `hashString`, `escapeHtml`/`escapeAttr`, `showToast`, `CONFIG` (Task 1), `store`, `registerRoute`/`matchRoute` (기존)
- Produces: `renderReceiptView(params)` (라우트 `/event/:id/receipt/:memberId`), `renderReceiptDom(ev, member, result)`, `renderBarcodeSvg(seed)`. 이 태스크는 사진 첨부/사진 관리/저장/공유/계좌복사 버튼을 **placeholder**(`showToast`)로만 연결한다 — Task 4/5/6이 각각 실제 동작으로 교체한다.

- [ ] **Step 1: 라우터에 영수증 라우트 추가**

Find:
```javascript
    const resultMatch = hash.match(/^\/event\/([^/]+)\/result$/);
    if (resultMatch && routes['/event/:id/result']) {
      return { render: routes['/event/:id/result'], params: { id: resultMatch[1] } };
    }
    return null;
  }
```

Replace with:
```javascript
    const resultMatch = hash.match(/^\/event\/([^/]+)\/result$/);
    if (resultMatch && routes['/event/:id/result']) {
      return { render: routes['/event/:id/result'], params: { id: resultMatch[1] } };
    }
    const receiptMatch = hash.match(/^\/event\/([^/]+)\/receipt\/([^/]+)$/);
    if (receiptMatch && routes['/event/:id/receipt/:memberId']) {
      return { render: routes['/event/:id/receipt/:memberId'], params: { id: receiptMatch[1], memberId: receiptMatch[2] } };
    }
    return null;
  }
```

- [ ] **Step 2: 정산 결과 화면의 영수증 버튼을 실제 라우팅으로 교체**

Find:
```javascript
    document.querySelectorAll('.btn-receipt').forEach(btn => {
      btn.addEventListener('click', () => { showToast('Stage 3에서 열립니다'); });
    });
```

Replace with:
```javascript
    document.querySelectorAll('.btn-receipt').forEach(btn => {
      btn.addEventListener('click', () => { location.hash = `#/event/${ev.id}/receipt/${btn.dataset.id}`; });
    });
```

- [ ] **Step 3: CSS 추가**

Find:
```css
  .account-info-box {
    display: flex;
    align-items: center;
    justify-content: space-between;
    background: #F9F4EC;
    border-radius: 10px;
    padding: 10px 12px;
    font-size: 14px;
    gap: 8px;
  }

  /* ====== 반응형 보강 ====== */
```

Replace with:
```css
  .account-info-box {
    display: flex;
    align-items: center;
    justify-content: space-between;
    background: #F9F4EC;
    border-radius: 10px;
    padding: 10px 12px;
    font-size: 14px;
    gap: 8px;
  }

  /* ====== 영수증 화면 ====== */
  .receipt-toolbar {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-bottom: 12px;
  }
  .receipt-member-tabs {
    display: flex;
    gap: 6px;
    overflow-x: auto;
    flex: 1;
    padding-bottom: 4px;
  }
  .receipt-tab {
    flex-shrink: 0;
    width: 40px;
    height: 40px;
    border-radius: 50%;
    border: 2px solid transparent;
    background: #F0E8DC;
    font-size: 18px;
    cursor: pointer;
  }
  .receipt-tab.active {
    border-color: var(--color-primary, #FF7A45);
    background: #fff;
  }
  .receipt-stage {
    background: #2B2420;
    border-radius: 16px;
    padding: 24px 12px;
    display: flex;
    justify-content: center;
    overflow-x: auto;
  }
  .receipt {
    width: 360px;
    flex-shrink: 0;
    background: #fdfdf8;
    color: #2A231C;
    font-family: 'Galmuri11', monospace;
    box-shadow: 0 12px 40px rgba(0,0,0,0.5);
  }
  .receipt-body { padding: 16px 20px; }
  .receipt-tear {
    height: 10px;
    background-color: #fdfdf8;
    background-image:
      linear-gradient(135deg, #2B2420 8px, transparent 0),
      linear-gradient(225deg, #2B2420 8px, transparent 0);
    background-size: 16px 16px;
    background-position: 0 0;
    background-repeat: repeat-x;
  }
  .receipt-tear-bottom { transform: scaleY(-1); }
  .receipt-shop { text-align: center; }
  .receipt-shop-name { font-size: 18px; font-weight: 700; }
  .receipt-shop-branch, .receipt-shop-owner { font-size: 12px; margin-top: 2px; }
  .receipt-divider { border-top: 1px dashed #B8AA96; margin: 10px 0; }
  .receipt-divider-bold { border-top: 2px dashed #2A231C; }
  .receipt-meta { font-size: 12px; display: flex; flex-direction: column; gap: 2px; }
  .receipt-photo-slot {
    margin: 4px 0;
    border-radius: 4px;
    overflow: hidden;
    cursor: pointer;
  }
  .receipt-photo-img { width: 100%; display: block; }
  .receipt-photo-empty {
    border: 2px dashed #B8AA96;
    border-radius: 4px;
    padding: 32px 0;
    text-align: center;
    font-size: 13px;
    color: #8A7A66;
  }
  .receipt-item-row {
    display: flex;
    align-items: baseline;
    gap: 4px;
    font-size: 13px;
    padding: 3px 0;
  }
  .receipt-item-name { white-space: nowrap; }
  .receipt-item-leader {
    flex: 1;
    border-bottom: 1px dotted #B8AA96;
    margin-bottom: 3px;
  }
  .receipt-item-amount {
    white-space: nowrap;
    font-variant-numeric: tabular-nums;
  }
  .receipt-footnote { font-size: 11px; color: #8A7A66; margin-top: 4px; }
  .receipt-total { text-align: center; }
  .receipt-total-row {
    display: flex;
    justify-content: space-between;
    font-size: 18px;
    font-weight: 700;
  }
  .receipt-stamp {
    display: inline-block;
    border: 3px solid #D64545;
    color: #D64545;
    padding: 6px 16px;
    font-size: 16px;
    font-weight: 700;
    transform: rotate(-8deg);
    margin: 8px auto;
  }
  .receipt-account {
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 12px;
    gap: 8px;
  }
  .receipt-barcode { text-align: center; margin-top: 12px; }
  .receipt-barcode-svg { max-width: 100%; }
  .receipt-barcode-id { font-size: 10px; color: #8A7A66; margin-top: 2px; }
  .receipt-ment {
    text-align: center;
    font-size: 11px;
    color: #8A7A66;
    margin-top: 10px;
    line-height: 1.6;
  }
  .receipt-actions {
    display: flex;
    gap: 8px;
    margin-top: 12px;
  }
  .receipt-actions .btn { flex: 1; }

  /* ====== 반응형 보강 ====== */
```

주의: `.receipt-tear`의 지그재그 절취선은 근사치 구현이다. 실제 브라우저에서 각도/타일 크기가 의도대로 보이지 않으면 Task 7의 폴리시 단계에서 `background-size`/그라디언트 각도를 조정한다 (jsdom으로는 시각 확인 불가).

- [ ] **Step 4: `renderReceiptDom` + `renderBarcodeSvg` + `renderReceiptView` 추가**

Find:
```javascript
  // ====== 라우트 등록 & 초기 실행 ======
```

Replace with:
```javascript
  // ====== 화면: 영수증 ======
  function renderBarcodeSvg(seed) {
    const barCount = 40;
    let bars = '';
    let x = 0;
    let s = seed;
    for (let i = 0; i < barCount; i++) {
      s = (s * 1103515245 + 12345) >>> 0; // 결정론적 의사난수 (선형합동법)
      const width = 1 + (s % 3);
      if ((s >> 8) % 2 === 0) {
        bars += `<rect x="${x}" y="0" width="${width}" height="32" fill="#2A231C" />`;
      }
      x += width + 1;
    }
    return `<svg class="receipt-barcode-svg" width="${x}" height="32" viewBox="0 0 ${x} 32">${bars}</svg>`;
  }

  function renderReceiptDom(ev, member, result) {
    const owner = ev.members.find(m => m.isOwner);
    const isOwner = member.isOwner === true;
    const memberIndex = ev.members.indexOf(member);
    const receiptNo = `${ev.id.slice(0, 8).toUpperCase()}-${String(memberIndex + 1).padStart(2, '0')}`;
    const issuedAt = new Date().toLocaleTimeString(CONFIG.DATE_LOCALE, { hour: '2-digit', minute: '2-digit' });
    const seed = hashString(ev.id + member.id);

    const items = ev.items
      .filter(item => result.matrix[item.id] && result.matrix[item.id][member.id] !== undefined)
      .slice()
      .sort((a, b) => a.order - b.order);
    const hasFixed = items.some(item => item.fixedAmounts && item.fixedAmounts[member.id] !== undefined);

    const photoDataUrl = photoStore.get(ev.id, member.id);
    const hasAccount = !isOwner && (ev.account.bank || ev.account.number || ev.account.holder);
    const ment = CONFIG.RECEIPT_MENTS[seed % CONFIG.RECEIPT_MENTS.length];

    return `
      <div class="receipt" id="receipt-capture" data-member-id="${member.id}">
        <div class="receipt-tear receipt-tear-top"></div>
        <div class="receipt-body">
          <div class="receipt-shop">
            <div class="receipt-shop-name">${escapeHtml(CONFIG.APP_NAME)}</div>
            <div class="receipt-shop-branch">${escapeHtml(ev.name)}점</div>
            <div class="receipt-shop-owner">대표: ${owner ? escapeHtml(owner.name) : '-'}</div>
          </div>
          <div class="receipt-divider"></div>
          <div class="receipt-meta">
            <div>${escapeHtml(ev.date)}</div>
            <div>발행 ${issuedAt}</div>
            <div>NO. ${receiptNo}</div>
          </div>
          <div class="receipt-divider"></div>
          <div class="receipt-photo-slot" id="receipt-photo-slot" data-id="${member.id}">
            ${photoDataUrl
              ? `<img src="${photoDataUrl}" alt="하이라이트 사진" class="receipt-photo-img" />`
              : `<div class="receipt-photo-empty">📷 하이라이트 사진 추가</div>`}
          </div>
          <div class="receipt-divider"></div>
          <div class="receipt-items">
            ${items.map(item => {
              const amount = result.matrix[item.id][member.id];
              const fixed = item.fixedAmounts && item.fixedAmounts[member.id] !== undefined;
              return `
                <div class="receipt-item-row">
                  <span class="receipt-item-name">${item.emoji ? escapeHtml(item.emoji) + ' ' : ''}${escapeHtml(item.name)}</span>
                  <span class="receipt-item-leader"></span>
                  <span class="receipt-item-amount">₩${amount.toLocaleString(CONFIG.CURRENCY_LOCALE)}${fixed ? '*' : ''}</span>
                </div>
              `;
            }).join('')}
            ${hasFixed ? '<div class="receipt-footnote">* 직접 조정된 금액</div>' : ''}
          </div>
          <div class="receipt-divider receipt-divider-bold"></div>
          <div class="receipt-total">
            ${isOwner
              ? `<div class="receipt-stamp">결제 완료</div>`
              : `<div class="receipt-total-row"><span>합계</span><span>₩${(result.transfers[member.id] || 0).toLocaleString(CONFIG.CURRENCY_LOCALE)}</span></div>`}
          </div>
          ${hasAccount ? `
            <div class="receipt-divider"></div>
            <div class="receipt-account">
              <span>${escapeHtml(ev.account.bank)} ${escapeHtml(ev.account.number)} (${escapeHtml(ev.account.holder)})</span>
              <button type="button" class="btn btn-ghost" id="btn-receipt-copy-account" data-no-capture>복사</button>
            </div>
          ` : ''}
          <div class="receipt-barcode">
            ${renderBarcodeSvg(seed)}
            <div class="receipt-barcode-id">${ev.id.slice(0, 12)}</div>
          </div>
          <div class="receipt-ment">
            <div>${escapeHtml(ment)}</div>
            <div>⁂ ${escapeHtml(CONFIG.APP_SHORT)} ⁂</div>
          </div>
        </div>
        <div class="receipt-tear receipt-tear-bottom"></div>
      </div>
    `;
  }

  function renderReceiptView(params) {
    const app = document.getElementById('app');
    const ev = store.get(params.id);
    if (!ev) { location.hash = '#/'; return; }
    if (ev.members.length === 0) { location.hash = `#/event/${ev.id}/result`; return; }

    let member = ev.members.find(m => m.id === params.memberId);
    if (!member) { member = ev.members[0]; }

    const result = calcSettlement(ev);

    app.innerHTML = `
      <header class="topbar">
        <button type="button" class="btn-back" id="btn-back">←</button>
        <h1>영수증</h1>
      </header>
      <main class="screen">
        <div class="receipt-toolbar">
          <div class="receipt-member-tabs" id="receipt-member-tabs">
            ${ev.members.map(m => `<button type="button" class="receipt-tab${m.id === member.id ? ' active' : ''}" data-id="${m.id}">${m.emoji}</button>`).join('')}
          </div>
          <button type="button" id="btn-photo-manager" class="btn btn-ghost" title="사진 관리">🖼</button>
          <button type="button" id="btn-save-all" class="btn btn-ghost" title="전체 저장">⬇전체</button>
        </div>
        <div class="receipt-stage">
          ${renderReceiptDom(ev, member, result)}
        </div>
        <div class="receipt-actions">
          <button type="button" id="btn-save-receipt" class="btn btn-primary">이미지 저장</button>
          <button type="button" id="btn-share-receipt" class="btn btn-ghost">이미지 공유</button>
        </div>
      </main>
    `;

    document.getElementById('btn-back').addEventListener('click', () => { location.hash = `#/event/${ev.id}/result`; });

    document.querySelectorAll('.receipt-tab').forEach(tab => {
      tab.addEventListener('click', () => { location.hash = `#/event/${ev.id}/receipt/${tab.dataset.id}`; });
    });

    const photoSlot = document.getElementById('receipt-photo-slot');
    if (photoSlot) {
      photoSlot.addEventListener('click', () => { showToast('Stage 3 후속 태스크에서 사진 첨부가 연결됩니다'); });
    }
    document.getElementById('btn-photo-manager').addEventListener('click', () => { showToast('Stage 3 후속 태스크에서 연결됩니다'); });
    document.getElementById('btn-save-all').addEventListener('click', () => { showToast('Stage 3 후속 태스크에서 연결됩니다'); });
    document.getElementById('btn-save-receipt').addEventListener('click', () => { showToast('Stage 3 후속 태스크에서 연결됩니다'); });
    document.getElementById('btn-share-receipt').addEventListener('click', () => { showToast('Stage 3 후속 태스크에서 연결됩니다'); });
    const copyBtn = document.getElementById('btn-receipt-copy-account');
    if (copyBtn) {
      copyBtn.addEventListener('click', () => { showToast('Stage 3 후속 태스크에서 연결됩니다'); });
    }
  }

  // ====== 라우트 등록 & 초기 실행 ======
```

- [ ] **Step 5: 라우트 등록**

Find:
```javascript
  registerRoute('/event/:id/result', renderSettlementResult);
  window.addEventListener('DOMContentLoaded', router);
```

Replace with:
```javascript
  registerRoute('/event/:id/result', renderSettlementResult);
  registerRoute('/event/:id/receipt/:memberId', renderReceiptView);
  window.addEventListener('DOMContentLoaded', router);
```

- [ ] **Step 6: 검증**

Node+jsdom(Task 1과 동일한 방식)으로 실제 UI 흐름 구동 가능: 이벤트 생성 → 시드 A 유사 데이터(멤버 5인, 항목 여러 개, 그중 하나는 고정 금액) 입력 → `#/event/:id/receipt/:memberId`로 이동 → `document.querySelector('.receipt-item-row')` 개수와 `document.querySelector('.receipt-total-row')` 텍스트가 `calcSettlement` 결과와 일치하는지 확인, 탭 클릭 시 `location.hash`가 바뀌고 다른 멤버 데이터로 재렌더되는지 확인, 총무 탭에서는 `.receipt-stamp`가 보이고 `.receipt-account`가 없는지 확인.

지그재그 절취선/바코드 SVG/도트 폰트의 실제 시각 품질은 jsdom으로 확인할 수 없다 — 가능하면 Chrome에서 스크린샷으로 재확인하고, 불가능하면 Task 7 완료 기준 검증 시 다시 시도.

- [ ] **Step 7: 커밋**

```bash
git add index.html
git commit -m "Stage 3: 라우터 확장 + 영수증 화면 전체(탭·8섹션 DOM·CSS)"
```

---

### Task 4: 사진 첨부 모달

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `resizeImageToCanvas`/`applyPhotoFilter`/`canvasToCompressedDataURL` (Task 2), `photoStore` (Task 1), `confirmDialog`, `renderReceiptView` (Task 3)
- Produces: `openPhotoAttachModal(eventId, memberId)`. Task 3의 사진 슬롯 placeholder를 실제 호출로 교체. **주의**: 이 태스크의 확정 핸들러는 `QuotaExceededError` 발생 시 `openPhotoManagerSheet`를 호출하지만 그 함수는 Task 5에서 정의된다 — 이 태스크의 검증에서는 용량 초과 경로를 실제로 트리거하지 않는다(해당 케이스는 Task 5 검증에서 다룬다).

- [ ] **Step 1: CSS 추가 (모달 오버레이/필터 미리보기)**

Find:
```css
  .receipt-actions .btn { flex: 1; }

  /* ====== 반응형 보강 ====== */
```

Replace with:
```css
  .receipt-actions .btn { flex: 1; }

  .sheet-overlay {
    position: fixed;
    inset: 0;
    background: rgba(0,0,0,0.4);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 100;
  }
  .sheet-box {
    background: #fff;
    border-radius: 16px;
    padding: 20px;
    width: min(400px, 92vw);
    max-height: 85vh;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }
  .sheet-actions { display: flex; gap: 8px; justify-content: flex-end; }
  .photo-filter-preview { display: flex; gap: 12px; justify-content: center; }
  .photo-filter-option { display: flex; flex-direction: column; align-items: center; gap: 4px; cursor: pointer; font-size: 12px; }
  .photo-filter-thumb-wrap {
    width: 72px;
    height: 72px;
    border-radius: 8px;
    overflow: hidden;
    border: 2px solid transparent;
    display: flex;
    align-items: center;
    justify-content: center;
    background: #F0E8DC;
  }
  .photo-filter-option.active .photo-filter-thumb-wrap { border-color: var(--color-primary, #FF7A45); }
  .photo-filter-thumb { width: 100%; height: 100%; object-fit: cover; }

  /* ====== 반응형 보강 ====== */
```

- [ ] **Step 2: `openPhotoAttachModal` 추가**

Find:
```javascript
  // ====== 라우트 등록 & 초기 실행 ======
```

Replace with:
```javascript
  function showPhotoNoticeOnce() {
    return new Promise((resolve) => {
      if (localStorage.getItem(CONFIG.PHOTO_NOTICE_KEY)) { resolve(); return; }
      const overlay = document.createElement('div');
      overlay.className = 'dialog-overlay';
      overlay.innerHTML = `
        <div class="dialog-box">
          <p class="dialog-message">사진은 이 기기에만 저장돼요. 정산 내역은 어디서든 볼 수 있지만, 사진이 담긴 영수증은 이 기기에서 만들어 이미지로 공유해주세요.</p>
          <div class="dialog-actions">
            <button type="button" class="btn btn-primary" data-action="confirm">확인</button>
          </div>
        </div>
      `;
      overlay.addEventListener('click', (e) => {
        if (e.target.dataset.action === 'confirm' || e.target === overlay) {
          localStorage.setItem(CONFIG.PHOTO_NOTICE_KEY, '1');
          overlay.remove();
          resolve();
        }
      });
      document.body.appendChild(overlay);
    });
  }

  let pendingPhotoCanvas = null;
  let pendingPhotoFilter = 'original';

  async function openPhotoAttachModal(eventId, memberId) {
    await showPhotoNoticeOnce();

    const overlay = document.createElement('div');
    overlay.className = 'sheet-overlay';
    const hasExisting = !!photoStore.get(eventId, memberId);
    overlay.innerHTML = `
      <div class="sheet-box">
        <h2>하이라이트 사진</h2>
        ${hasExisting ? '<button type="button" id="btn-remove-photo" class="btn btn-danger" style="width:100%">현재 사진 삭제</button>' : ''}
        <input type="file" accept="image/*" capture id="photo-file-input" class="input" />
        <div id="photo-filter-preview" class="photo-filter-preview"></div>
        <div class="sheet-actions">
          <button type="button" id="btn-photo-cancel" class="btn btn-ghost">취소</button>
          <button type="button" id="btn-photo-confirm" class="btn btn-primary" disabled>확정</button>
        </div>
      </div>
    `;
    document.body.appendChild(overlay);

    pendingPhotoCanvas = null;
    pendingPhotoFilter = 'original';

    function closeModal() {
      pendingPhotoCanvas = null;
      overlay.remove();
    }

    overlay.addEventListener('click', (e) => { if (e.target === overlay) closeModal(); });
    document.getElementById('btn-photo-cancel').addEventListener('click', closeModal);

    const removeBtn = document.getElementById('btn-remove-photo');
    if (removeBtn) {
      removeBtn.addEventListener('click', async () => {
        const ok = await confirmDialog('사진을 삭제할까요?');
        if (ok) {
          photoStore.remove(eventId, memberId);
          closeModal();
          renderReceiptView({ id: eventId, memberId });
        }
      });
    }

    function renderPhotoFilterPreview() {
      const previewEl = document.getElementById('photo-filter-preview');
      if (!previewEl) return;
      if (!pendingPhotoCanvas) {
        previewEl.innerHTML = '';
        document.getElementById('btn-photo-confirm').disabled = true;
        return;
      }
      const filters = ['original', 'grayscale', 'dither'];
      const labels = { original: '원본 컬러', grayscale: '흑백', dither: '도트' };
      previewEl.innerHTML = filters.map(f => `
        <div class="photo-filter-option${f === pendingPhotoFilter ? ' active' : ''}" data-filter="${f}">
          <div class="photo-filter-thumb-wrap"></div>
          <span>${labels[f]}</span>
        </div>
      `).join('');
      filters.forEach(f => {
        const wrap = previewEl.querySelector(`.photo-filter-option[data-filter="${f}"] .photo-filter-thumb-wrap`);
        const filteredCanvas = applyPhotoFilter(pendingPhotoCanvas, f);
        filteredCanvas.className = 'photo-filter-thumb';
        wrap.appendChild(filteredCanvas);
      });
      previewEl.querySelectorAll('.photo-filter-option').forEach(opt => {
        opt.addEventListener('click', () => {
          pendingPhotoFilter = opt.dataset.filter;
          renderPhotoFilterPreview();
        });
      });
      document.getElementById('btn-photo-confirm').disabled = false;
    }

    document.getElementById('photo-file-input').addEventListener('change', async (e) => {
      const file = e.target.files[0];
      if (!file) return;
      pendingPhotoCanvas = await resizeImageToCanvas(file, CONFIG.PHOTO_MAX_EDGE);
      pendingPhotoFilter = 'original';
      renderPhotoFilterPreview();
    });

    document.getElementById('btn-photo-confirm').addEventListener('click', () => {
      if (!pendingPhotoCanvas) return;
      const finalCanvas = applyPhotoFilter(pendingPhotoCanvas, pendingPhotoFilter);
      const dataUrl = canvasToCompressedDataURL(finalCanvas, CONFIG.PHOTO_QUALITY);
      try {
        photoStore.set(eventId, memberId, dataUrl);
      } catch (err) {
        if (err.name === 'QuotaExceededError') {
          closeModal();
          showToast('사진 용량이 가득 찼어요. 지난 모임 사진을 정리해주세요');
          openPhotoManagerSheet(eventId);
          return;
        }
        throw err;
      }
      closeModal();
      renderReceiptView({ id: eventId, memberId });
    });
  }

  // ====== 라우트 등록 & 초기 실행 ======
```

- [ ] **Step 3: Task 3의 사진 슬롯 placeholder를 실제 호출로 교체**

Find:
```javascript
    const photoSlot = document.getElementById('receipt-photo-slot');
    if (photoSlot) {
      photoSlot.addEventListener('click', () => { showToast('Stage 3 후속 태스크에서 사진 첨부가 연결됩니다'); });
    }
```

Replace with:
```javascript
    const photoSlot = document.getElementById('receipt-photo-slot');
    if (photoSlot) {
      photoSlot.addEventListener('click', () => { openPhotoAttachModal(ev.id, member.id); });
    }
```

- [ ] **Step 4: 검증 (실제 브라우저 필요 — 파일 선택 UI와 canvas 렌더링이 관여)**

`http://localhost:8123/index.html`에서 영수증 화면 진입 → 사진 슬롯 탭 → 최초 1회 안내 모달 표시 확인 → 확인 클릭 → 파일 선택 → 3개 필터 썸네일이 서로 다르게 보이는지 확인 → "흑백" 선택 → 확정 → 슬롯에 사진이 표시되는지 + `photoStore.get(evId, memberId)`가 `data:image/jpeg;base64,...`로 채워졌는지 콘솔에서 확인 → 두 번째 안내 모달이 다시 뜨지 않는지 확인(같은 브라우저 세션에서 재시도) → "현재 사진 삭제" 동작 확인.

Chrome 자동화가 불가능하면 이 태스크는 코드 리뷰(정적 분석)로만 확인하고 최종 보고서에 "실브라우저 미검증"으로 명시, Task 7에서 재시도.

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "Stage 3: 사진 첨부 모달(1회 안내 · 파일선택 · 필터 미리보기 · 확정/삭제)"
```

---

### Task 5: 사진 관리 시트 + 용량 초과 연동 + 모임 삭제 시 사진 정리

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `photoStore` (Task 1), `confirmDialog`/`showToast`, `store.get`, `onDeleteEvent` (기존, Stage 1)
- Produces: `openPhotoManagerSheet(eventId)`. Task 3의 "사진 관리" 버튼 placeholder를 실제 호출로 교체. `onDeleteEvent`에 `photoStore.removeByEvent` 훅 추가.

- [ ] **Step 1: CSS 추가**

Find:
```css
  .photo-filter-thumb { width: 100%; height: 100%; object-fit: cover; }

  /* ====== 반응형 보강 ====== */
```

Replace with:
```css
  .photo-filter-thumb { width: 100%; height: 100%; object-fit: cover; }

  .photo-usage-hint { font-size: 12px; color: #8A7A66; }
  .photo-manager-grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 10px;
  }
  .photo-manager-item {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 4px;
    font-size: 11px;
  }
  .photo-manager-item img {
    width: 100%;
    aspect-ratio: 1;
    object-fit: cover;
    border-radius: 8px;
  }

  /* ====== 반응형 보강 ====== */
```

- [ ] **Step 2: `openPhotoManagerSheet` 추가**

Find:
```javascript
  // ====== 라우트 등록 & 초기 실행 ======
```

Replace with:
```javascript
  function openPhotoManagerSheet(eventId) {
    const ev = store.get(eventId);
    const overlay = document.createElement('div');
    overlay.className = 'sheet-overlay';
    const photos = photoStore.listByEvent(eventId);
    overlay.innerHTML = `
      <div class="sheet-box">
        <h2>사진 관리</h2>
        <div class="photo-usage-hint">전체 사진 용량: 약 ${photoStore.usage()}KB</div>
        <div class="photo-manager-grid" id="photo-manager-grid">
          ${photos.length === 0 ? '<div class="field-hint">첨부된 사진이 없어요</div>' : photos.map(p => {
            const m = ev ? ev.members.find(mm => mm.id === p.memberId) : null;
            return `
              <div class="photo-manager-item" data-member-id="${p.memberId}">
                <img src="${p.dataUrl}" alt="${m ? escapeAttr(m.name) : ''}" />
                <span>${m ? escapeHtml(m.name) : '(삭제된 멤버)'}</span>
                <button type="button" class="btn btn-danger photo-manager-remove" data-member-id="${p.memberId}">삭제</button>
              </div>
            `;
          }).join('')}
        </div>
        ${photos.length > 0 ? '<button type="button" id="btn-remove-all-photos" class="btn btn-danger" style="width:100%">이 모임 사진 전체 삭제</button>' : ''}
        <div class="sheet-actions">
          <button type="button" id="btn-photo-manager-close" class="btn btn-ghost">닫기</button>
        </div>
      </div>
    `;
    document.body.appendChild(overlay);

    function close() { overlay.remove(); }
    overlay.addEventListener('click', (e) => { if (e.target === overlay) close(); });
    document.getElementById('btn-photo-manager-close').addEventListener('click', close);

    overlay.querySelectorAll('.photo-manager-remove').forEach(btn => {
      btn.addEventListener('click', async () => {
        const ok = await confirmDialog('이 사진을 삭제할까요?');
        if (ok) {
          photoStore.remove(eventId, btn.dataset.memberId);
          close();
          openPhotoManagerSheet(eventId);
        }
      });
    });

    const removeAllBtn = document.getElementById('btn-remove-all-photos');
    if (removeAllBtn) {
      removeAllBtn.addEventListener('click', async () => {
        const ok = await confirmDialog('이 모임의 모든 사진을 삭제할까요?');
        if (ok) {
          photoStore.removeByEvent(eventId);
          close();
          showToast('사진을 모두 삭제했어요');
        }
      });
    }
  }

  // ====== 라우트 등록 & 초기 실행 ======
```

- [ ] **Step 3: Task 3의 "사진 관리" 버튼 placeholder를 실제 호출로 교체**

Find:
```javascript
    document.getElementById('btn-photo-manager').addEventListener('click', () => { showToast('Stage 3 후속 태스크에서 연결됩니다'); });
```

Replace with:
```javascript
    document.getElementById('btn-photo-manager').addEventListener('click', () => { openPhotoManagerSheet(ev.id); });
```

- [ ] **Step 4: 모임 삭제 시 사진 정리 훅 추가**

Find:
```javascript
  async function onDeleteEvent(id) {
    const ev = store.get(id);
    if (!ev) return;
    const ok = await confirmDialog(`"${ev.name}" 모임을 삭제할까요?`);
    if (ok) {
      store.remove(id);
      renderHome();
    }
  }
```

Replace with:
```javascript
  async function onDeleteEvent(id) {
    const ev = store.get(id);
    if (!ev) return;
    const ok = await confirmDialog(`"${ev.name}" 모임을 삭제할까요?`);
    if (ok) {
      store.remove(id);
      photoStore.removeByEvent(id);
      renderHome();
    }
  }
```

- [ ] **Step 5: 검증**

**용량 초과 시뮬레이션** (Node+jsdom 또는 실제 브라우저 콘솔, 둘 다 가능 — `localStorage.setItem`을 일시적으로 오버라이드하는 순수 JS이므로 jsdom에서도 동작):
```javascript
const originalSetItem = localStorage.setItem.bind(localStorage);
localStorage.setItem = () => { throw new DOMException('quota exceeded', 'QuotaExceededError'); };
try {
  photoStore.set('evX', 'mX', 'data:image/jpeg;base64,AAAA');
  console.log('FAIL: 예외가 발생하지 않음');
} catch (err) {
  console.log('OK: 예외 이름 =', err.name); // 'QuotaExceededError' 예상
}
localStorage.setItem = originalSetItem;
```
이 스니펫으로 `photoStore.set`이 `QuotaExceededError`를 실제로 전파하는지 확인한다. 실 브라우저에서는 추가로 사진 첨부 모달의 확정 버튼 클릭 시 위 오버라이드가 걸린 상태에서 `showToast` + `openPhotoManagerSheet` 자동 오픈까지 이어지는지 확인.

**모임 삭제 연동**: 사진이 첨부된 모임을 하나 만들고 삭제 → `photoStore.listByEvent(evId).length === 0` 확인.

**사진 관리 시트**: `openPhotoManagerSheet(evId)` 직접 호출(콘솔) → 썸네일 그리드/용량 표시/개별 삭제/전체 삭제 버튼 동작 확인.

- [ ] **Step 6: 커밋**

```bash
git add index.html
git commit -m "Stage 3: 사진 관리 시트 + 용량 초과 처리 + 모임 삭제 시 사진 정리"
```

---

### Task 6: 캡처 · 공유 · 저장 파이프라인 + html2canvas 연동

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: html2canvas(CDN), `renderReceiptDom`/`renderReceiptView` (Task 3), `calcSettlement`, `showToast`
- Produces: `captureReceiptToBlob(receiptEl) → Promise<Blob>`, `downloadBlob(blob, filename)`, `saveReceiptImage(ev, member)`, `shareReceiptImage(ev, member)`, `saveAllReceipts(ev)`. Task 3의 남은 placeholder 버튼들(저장/공유/전체저장/계좌복사)을 실제 동작으로 교체.

- [ ] **Step 1: html2canvas CDN 스크립트 추가**

Find:
```html
<div id="app"></div>

<script>
```

Replace with:
```html
<div id="app"></div>

<script src="https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js"></script>
<script>
```

- [ ] **Step 2: 캡처·공유·저장 함수 추가**

Find:
```javascript
  // ====== 라우트 등록 & 초기 실행 ======
```

Replace with:
```javascript
  async function captureReceiptToBlob(receiptEl) {
    await document.fonts.ready;
    const canvas = await html2canvas(receiptEl, {
      scale: CONFIG.RECEIPT_SCALE,
      backgroundColor: null,
      ignoreElements: (el) => el.dataset && el.dataset.noCapture !== undefined,
    });
    return new Promise(resolve => canvas.toBlob(resolve, 'image/png'));
  }

  function downloadBlob(blob, filename) {
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = filename;
    document.body.appendChild(a);
    a.click();
    a.remove();
    URL.revokeObjectURL(url);
  }

  async function saveReceiptImage(ev, member) {
    const receiptEl = document.getElementById('receipt-capture');
    if (!receiptEl) return;
    const blob = await captureReceiptToBlob(receiptEl);
    downloadBlob(blob, `오내총_${ev.name}_${member.name}.png`);
    showToast('영수증 이미지를 저장했어요');
  }

  async function shareReceiptImage(ev, member) {
    const receiptEl = document.getElementById('receipt-capture');
    if (!receiptEl) return;
    const blob = await captureReceiptToBlob(receiptEl);
    const file = new File([blob], `오내총_${ev.name}_${member.name}.png`, { type: 'image/png' });
    if (navigator.canShare && navigator.canShare({ files: [file] })) {
      try {
        await navigator.share({ files: [file], title: `${ev.name} 영수증` });
        return;
      } catch (err) {
        if (err.name === 'AbortError') return; // 사용자가 공유를 취소함
      }
    }
    downloadBlob(blob, `오내총_${ev.name}_${member.name}.png`);
    showToast('갤러리에서 공유해주세요');
  }

  async function saveAllReceipts(ev) {
    const activeTab = document.querySelector('.receipt-tab.active');
    const originalMemberId = activeTab ? activeTab.dataset.id : ev.members[0].id;
    showToast(`영수증 저장 중... (0/${ev.members.length})`);
    const app = document.getElementById('app');
    for (let i = 0; i < ev.members.length; i++) {
      const member = ev.members[i];
      const result = calcSettlement(ev);
      const stage = app.querySelector('.receipt-stage');
      if (stage) { stage.innerHTML = renderReceiptDom(ev, member, result); }
      await new Promise(r => setTimeout(r, 150));
      await saveReceiptImage(ev, member);
    }
    renderReceiptView({ id: ev.id, memberId: originalMemberId });
    showToast('전체 영수증 저장을 완료했어요');
  }

  // ====== 라우트 등록 & 초기 실행 ======
```

- [ ] **Step 3: Task 3의 남은 placeholder들을 실제 호출로 교체**

Find:
```javascript
    document.getElementById('btn-save-all').addEventListener('click', () => { showToast('Stage 3 후속 태스크에서 연결됩니다'); });
    document.getElementById('btn-save-receipt').addEventListener('click', () => { showToast('Stage 3 후속 태스크에서 연결됩니다'); });
    document.getElementById('btn-share-receipt').addEventListener('click', () => { showToast('Stage 3 후속 태스크에서 연결됩니다'); });
    const copyBtn = document.getElementById('btn-receipt-copy-account');
    if (copyBtn) {
      copyBtn.addEventListener('click', () => { showToast('Stage 3 후속 태스크에서 연결됩니다'); });
    }
```

Replace with:
```javascript
    document.getElementById('btn-save-all').addEventListener('click', () => { saveAllReceipts(ev); });
    document.getElementById('btn-save-receipt').addEventListener('click', () => { saveReceiptImage(ev, member); });
    document.getElementById('btn-share-receipt').addEventListener('click', () => { shareReceiptImage(ev, member); });
    const copyBtn = document.getElementById('btn-receipt-copy-account');
    if (copyBtn) {
      copyBtn.addEventListener('click', () => {
        const text = `${ev.account.bank} ${ev.account.number} ${ev.account.holder}`.trim();
        navigator.clipboard.writeText(text).then(() => showToast('계좌 정보를 복사했어요'));
      });
    }
```

- [ ] **Step 4: 검증 (실제 브라우저 필요 — html2canvas 래스터라이즈, 파일 다운로드/Web Share API가 관여. jsdom은 `captureReceiptToBlob`이 예외 없이 Blob을 반환하고 `blob.size > 0`인지까지만 확인 가능)**

Node+jsdom (가능한 범위):
```javascript
// jsdom 환경에서 html2canvas 자체는 신뢰할 수 없는 rasterization을 반환할 수 있으므로,
// 여기서는 captureReceiptToBlob 호출이 에러 없이 완료되고 Blob 크기가 0이 아닌지만 확인한다.
const el = document.getElementById('receipt-capture');
captureReceiptToBlob(el).then(blob => console.log('blob size:', blob.size));
```

실제 브라우저(우선 시도): 영수증 화면에서 "이미지 저장" 클릭 → 다운로드된 PNG가 720px 폭(`RECEIPT_WIDTH(360) × RECEIPT_SCALE(2)`)인지, 도트 폰트가 깨지지 않는지, 계좌 복사 버튼 등 `data-no-capture` 요소가 이미지에 없는지 확인. "이미지 공유" 클릭 시 `navigator.canShare` 지원 환경이면 공유 시트가 뜨는지, 미지원(예: 데스크톱 Chrome)이면 자동 다운로드 + "갤러리에서 공유해주세요" 토스트가 뜨는지 확인. "전체 저장" 클릭 시 멤버 수만큼 순차 다운로드되고 마지막에 원래 보고 있던 멤버 영수증으로 화면이 복귀하는지 확인.

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "Stage 3: html2canvas 캡처·공유·저장·전체저장 파이프라인"
```

---

### Task 7: 완료 기준 8개 전체 검증 + CLAUDE.md 문서 업데이트

**Files:**
- Modify: `CLAUDE.md`
- (필요 시) Modify: `index.html` — 검증 중 발견된 버그 수정 (버그가 없다면 CSS/DOM은 그대로 유지)

**Interfaces:**
- Consumes: 전체 Stage 3 구현 (Task 1-6)
- Produces: 없음 (마감 태스크)

- [ ] **Step 1: 완료 기준 8개 순서대로 검증**

Chrome 브라우저 자동화를 먼저 시도한다. 가능하면 아래 모든 항목을 실제 브라우저에서 시각적으로 확인한다. 불가능하면(이번 세션의 Stage 2 작업 내내 연결 실패했음) Node+jsdom으로 가능한 항목만 확인하고, canvas/html2canvas가 관여하는 항목은 "미검증 — 실브라우저 필요"로 최종 보고서에 명시한다.

1. **시드 A 은찬 영수증**: 시드 A 데이터(5인, 10항목, 총액 ₩199,100) 입력 → 은찬 영수증 → 참여 항목 10개 + 합계 ₩39,820 표시 확인(jsdom 가능: `.receipt-item-row` 개수와 `.receipt-total-row` 텍스트). 도트 폰트·절취선·바코드 렌더링은 실브라우저 확인.
2. **사진 첨부 → 필터 → 저장**: 사진 첨부 → 3종 필터 미리보기 → 흑백 확정 → `photoStore.get(evId, memberId)`이 압축된 `data:image/jpeg` 문자열인지, 그리고 리사이즈된 원본(압축 전)이 `localStorage`에 별도로 남아있지 않은지 확인(실브라우저 필요 — 파일 선택 UI).
3. **PNG 저장**: 720px 폭, 폰트 안 깨짐, 복사 버튼 등 UI 요소 미포함 (실브라우저 필요).
4. **`navigator.share` 미지원 폴백**: 콘솔에서 `navigator.canShare = undefined` 임시 오버라이드 후 "이미지 공유" 클릭 → 자동 다운로드 + 토스트 확인(실브라우저, 다운로드 트리거 필요).
5. **총무 영수증**: 정우(총무) 영수증 → `.receipt-stamp`("결제 완료") 표시, `.receipt-account` 없음 (jsdom 가능).
6. **고정 금액 `*` 표시**: 고정 금액이 있는 항목이 포함된 멤버의 영수증에서 `*` 및 각주(`.receipt-footnote`) 노출 확인 (jsdom 가능).
7. **용량 초과 시뮬레이션**: Task 5 Step 5의 스니펫으로 재확인 → 안내 토스트 + 사진 관리 시트 동작 (jsdom 가능, UI 트리거는 실브라우저 권장).
8. **모임 삭제 시 사진 정리**: Task 5 Step 5에서 이미 확인 — 재확인만.

- [ ] **Step 2: `CLAUDE.md` "Project status" 업데이트**

Find:
```markdown
Stage 1 (skeleton: hash routing, CONFIG, event/member CRUD) and Stage 2 (item entry, settlement engine, result table) are implemented in `index.html`. Stages 3-5 (receipts, Firebase sync, PWA polish) are not yet implemented — see `오내총_PRD_v1.md` §9 and `docs/superpowers/specs/` for design docs, and `docs/superpowers/plans/` for implementation plans.
```

Replace with:
```markdown
Stage 1 (skeleton: hash routing, CONFIG, event/member CRUD), Stage 2 (item entry, settlement engine, result table), and Stage 3 (감성 영수증: photo attach, dot-font receipt DOM, html2canvas capture/share) are implemented in `index.html`. Stages 4-5 (Firebase sync, PWA polish) are not yet implemented — see `오내총_PRD_v1.md` §9 and `docs/superpowers/specs/` for design docs, and `docs/superpowers/plans/` for implementation plans.
```

- [ ] **Step 3: `CLAUDE.md`에 photoStore API·캡처 규칙 기록**

Find:
```markdown
## Data storage model (hybrid — important, don't collapse into one store)
```

Replace with:
```markdown
## Photo storage & receipt capture (Stage 3 implementation reference)

- `photoStore` (in `index.html`, parallel to `store`) is the only code path that touches `localStorage[CONFIG.PHOTOS_KEY]`. API: `get(eventId, memberId)`, `set(eventId, memberId, dataUrl)` (throws `QuotaExceededError` on failure — callers must catch it), `remove(eventId, memberId)`, `removeByEvent(eventId)`, `listByEvent(eventId)`, `usage()` (KB estimate). `event.*` objects never reference photos directly — the two are joined only by the `"{eventId}:{memberId}"` key convention.
- Photos are resized (`resizeImageToCanvas`, long edge capped at `CONFIG.PHOTO_MAX_EDGE`) and filtered (`applyPhotoFilter`: `'original'|'grayscale'|'dither'`) via `<canvas>` pixel manipulation at save time — never via CSS `filter`, which `html2canvas` does not reliably capture. Only the final filtered/compressed result is persisted; the resized-but-unfiltered intermediate lives only in the module-scope `pendingPhotoCanvas` variable during the attach modal's session and is never written to `localStorage`.
- Any element that must not appear in a captured receipt PNG (e.g. the account-copy button) must carry a `data-no-capture` attribute — `captureReceiptToBlob`'s `html2canvas` call excludes those via `ignoreElements`.
- Every capture (`captureReceiptToBlob`) must `await document.fonts.ready` first, or the Galmuri dot font may not have finished loading/rasterizing at capture time.
- `onDeleteEvent` calls `photoStore.removeByEvent(id)` immediately after `store.remove(id)` — deleting an event always cleans up its photos in the same action.

## Data storage model (hybrid — important, don't collapse into one store)
```

- [ ] **Step 4: 커밋**

```bash
git add index.html CLAUDE.md
git commit -m "Stage 3: 완료 기준 8개 검증 + CLAUDE.md photoStore/캡처 규칙 기록"
```

---

## Self-Review 결과 (계획 작성자 자체 점검)

- **스펙 커버리지**: 디자인 스펙 §2~§13 각 섹션에 대응하는 태스크 존재 확인 — §2 CONFIG(Task1), §3 photoStore(Task1), §4 필터파이프라인(Task2), §5~7 화면/DOM/CSS(Task3), §8 첨부모달(Task4), §9 관리시트(Task5), §10 캡처/공유(Task6), §11 삭제연동(Task5), §13 검증방식(Task7 Step1). 누락 없음.
- **placeholder 스캔**: "TBD"/"TODO" 없음. Task 3의 `showToast('Stage 3 후속 태스크에서 연결됩니다')`는 의도된 임시 자리표시자이며 Task 4/5/6이 명시적으로 교체하는 "Find/Replace" 스텝을 갖고 있음 — 방치되는 placeholder 아님.
- **타입/시그니처 일관성**: `renderReceiptView(params)`의 `params`는 Task 3~6에서 항상 `{id, memberId}` 형태로 일관되게 전달됨. `photoStore`의 메서드명(`get/set/remove/removeByEvent/listByEvent/usage`)이 Task 1 정의 이후 모든 소비 태스크(3,4,5,6)에서 동일하게 사용됨. `applyPhotoFilter(canvas, filterName)`의 `filterName` 값(`'original'|'grayscale'|'dither'`)이 Task 2 정의와 Task 4의 모달 UI 문자열이 일치함.
