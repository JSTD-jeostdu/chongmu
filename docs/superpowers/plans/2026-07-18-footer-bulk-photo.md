# 제작자 푸터 + 사진 일괄 삽입 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `index.html`에 (1) 모든 라우트 최하단에 항상 노출되는 정적 제작자 푸터를 추가하고, (2) 사진 관리 시트에서 이벤트의 전체 멤버에게 같은 하이라이트 사진을 한 번에 적용하는 기능을 추가한다.

**Architecture:** 단일 파일(`index.html`) 내 추가. 푸터는 라우터가 건드리지 않는 `#app` 형제 위치에 정적 HTML/CSS로 추가한다. 사진 일괄 삽입은 기존 `openPhotoAttachModal`과 동일한 UI 패턴(파일 선택 → 필터 프리뷰 → 확정)을 재사용하는 새 함수 `openBulkPhotoAttachModal`을 추가하고, `openPhotoManagerSheet`에서 진입 버튼을 연결한다. 기존 `photoStore`, `resizeImageToCanvas`, `applyPhotoFilter`, `canvasToCompressedDataURL`, `confirmDialog`, `showToast`를 그대로 재사용하며 새 데이터 계층은 만들지 않는다.

**Tech Stack:** 변경 없음 (Vanilla JS/HTML/CSS, 신규 CDN 의존성 없음).

## Global Constraints

- 단일 `index.html` 파일 내에서만 작업한다. 빌드 도구·npm·번들러·프레임워크 없음.
- 신규 CDN 의존성을 추가하지 않는다.
- 주석은 한국어로 작성한다 (WHY가 비자명할 때만, 최소한으로).
- 새 리터럴 색상값은 `CONFIG`에 넣지 않는다 — Stage 1~3 선례를 따라 `CONFIG`에는 핵심 테마 색상 3종만 유지한다.
- 사진은 `photoStore`를 통해서만 `localStorage[CONFIG.PHOTOS_KEY]`에 접근한다. `event.*` 객체는 사진을 직접 참조하지 않는다.
- `photoStore.set`은 `QuotaExceededError`를 던질 수 있으므로 호출부는 반드시 catch한다.
- 이 저장소는 Node 테스트 러너가 없다 — 유일한 자동 검증은 `CONFIG.DEV_MODE=true`일 때 브라우저 콘솔에서 도는 `runRegressionTests`이며, 이번 변경은 그 스위트가 다루는 순수 데이터 로직(설산/포토스토어 CRUD)을 건드리지 않으므로 새 회귀 테스트는 추가하지 않는다. 각 태스크의 검증은 브라우저에서 파일을 직접 열어 수동으로 확인한다.

---

### Task 1: 제작자 푸터 추가

**Files:**
- Modify: `index.html` (`<style>` 블록 끝부분, `<body>` 시작부)

**Interfaces:**
- Consumes: 없음 (정적 마크업/CSS만 추가)
- Produces: `#app-footer` DOM 요소 (모든 라우트에서 상시 노출)

- [ ] **Step 1: 푸터 CSS 추가**

Find (`index.html` 내, `.photo-manager-item img` 규칙과 반응형 보강 섹션 사이):
```css
  .photo-manager-item img {
    width: 100%;
    aspect-ratio: 1;
    object-fit: cover;
    border-radius: 8px;
  }

  /* ====== 반응형 보강 ====== */
```

Replace with:
```css
  .photo-manager-item img {
    width: 100%;
    aspect-ratio: 1;
    object-fit: cover;
    border-radius: 8px;
  }

  /* ====== 제작자 푸터 ====== */
  #app-footer {
    max-width: 560px;
    margin: 32px auto 0;
    padding: 20px 16px 32px;
    text-align: center;
    font-size: 12px;
    line-height: 1.7;
    color: #B0A290;
  }
  #app-footer .app-footer-title {
    font-weight: 600;
    color: #8A7A66;
  }

  /* ====== 반응형 보강 ====== */
```

- [ ] **Step 2: 푸터 마크업을 `#app`과 형제 위치에 추가**

Find:
```html
<body>
<div id="app"></div>

<script src="https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js"></script>
```

Replace with:
```html
<body>
<div id="app"></div>
<footer id="app-footer">
  <div class="app-footer-title">오늘은 내가 총무</div>
  <div>made by JSTD</div>
</footer>

<script src="https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js"></script>
```

- [ ] **Step 3: 브라우저에서 수동 검증**

`index.html`을 브라우저로 직접 연다 (더블클릭 또는 `start index.html`). 확인 사항:
1. 홈 화면(`#/`)에서 스크롤을 내리면 화면 최하단에 "오늘은 내가 총무" / "made by JSTD" 두 줄이 보인다.
2. 이벤트 상세, 정산 결과, 영수증 화면 등 다른 라우트로 이동해도 동일하게 최하단에 푸터가 유지된다.
3. 영수증 화면에서 "이미지 저장"을 눌러 캡처된 PNG를 확인 — 캡처는 `#receipt-capture` 요소만 대상으로 하므로 푸터가 이미지에 포함되지 않는다.

- [ ] **Step 4: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
웹앱 최하단에 제작자 푸터 추가

EOF
)"
```

---

### Task 2: 사진 일괄 삽입 기능

**Files:**
- Modify: `index.html` (`openPhotoManagerSheet` 함수, 그 직후에 새 함수 `openBulkPhotoAttachModal` 추가)

**Interfaces:**
- Consumes: `photoStore.{get,set}`, `resizeImageToCanvas(file, maxEdge)`, `applyPhotoFilter(canvas, filterName)`, `canvasToCompressedDataURL(canvas, quality)`, `confirmDialog(message) → Promise<boolean>`, `showToast(message)`, `showPhotoNoticeOnce()`, `CONFIG.PHOTO_MAX_EDGE`, `CONFIG.PHOTO_QUALITY`, 모듈 전역 `pendingPhotoCanvas`/`pendingPhotoFilter` (기존 `openPhotoAttachModal`과 공유)
- Produces: `openBulkPhotoAttachModal(eventId)` — 사진 관리 시트의 새 버튼에서 호출됨

- [ ] **Step 1: 사진 관리 시트에 일괄 적용 버튼 추가**

Find:
```javascript
        ${photos.length > 0 ? '<button type="button" id="btn-remove-all-photos" class="btn btn-danger" style="width:100%">이 모임 사진 전체 삭제</button>' : ''}
        <div class="sheet-actions">
          <button type="button" id="btn-photo-manager-close" class="btn btn-ghost">닫기</button>
        </div>
      </div>
    `;
```

Replace with:
```javascript
        ${ev && ev.members.length > 0 ? '<button type="button" id="btn-bulk-photo-apply" class="btn btn-primary" style="width:100%">전체 멤버에게 같은 사진 적용</button>' : ''}
        ${photos.length > 0 ? '<button type="button" id="btn-remove-all-photos" class="btn btn-danger" style="width:100%">이 모임 사진 전체 삭제</button>' : ''}
        <div class="sheet-actions">
          <button type="button" id="btn-photo-manager-close" class="btn btn-ghost">닫기</button>
        </div>
      </div>
    `;
```

- [ ] **Step 2: 일괄 적용 버튼에 이벤트 리스너 연결**

Find:
```javascript
    function close() { overlay.remove(); }
    overlay.addEventListener('click', (e) => { if (e.target === overlay) close(); });
    document.getElementById('btn-photo-manager-close').addEventListener('click', close);

    overlay.querySelectorAll('.photo-manager-remove').forEach(btn => {
```

Replace with:
```javascript
    function close() { overlay.remove(); }
    overlay.addEventListener('click', (e) => { if (e.target === overlay) close(); });
    document.getElementById('btn-photo-manager-close').addEventListener('click', close);

    const bulkApplyBtn = document.getElementById('btn-bulk-photo-apply');
    if (bulkApplyBtn) {
      bulkApplyBtn.addEventListener('click', () => {
        close();
        openBulkPhotoAttachModal(eventId);
      });
    }

    overlay.querySelectorAll('.photo-manager-remove').forEach(btn => {
```

- [ ] **Step 3: `openBulkPhotoAttachModal` 함수 추가**

Find:
```javascript
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

  async function captureReceiptToBlob(receiptEl) {
```

Replace with:
```javascript
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

  async function openBulkPhotoAttachModal(eventId) {
    await showPhotoNoticeOnce();
    const ev = store.get(eventId);
    if (!ev || ev.members.length === 0) return;

    const overlay = document.createElement('div');
    overlay.className = 'sheet-overlay';
    overlay.innerHTML = `
      <div class="sheet-box">
        <h2>전체 멤버에게 같은 사진 적용</h2>
        <input type="file" accept="image/*" id="bulk-photo-file-input" class="input" />
        <div id="bulk-photo-filter-preview" class="photo-filter-preview"></div>
        <div class="sheet-actions">
          <button type="button" id="btn-bulk-photo-cancel" class="btn btn-ghost">취소</button>
          <button type="button" id="btn-bulk-photo-confirm" class="btn btn-primary" disabled>전체 적용</button>
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
    document.getElementById('btn-bulk-photo-cancel').addEventListener('click', closeModal);

    function renderBulkPhotoFilterPreview() {
      const previewEl = document.getElementById('bulk-photo-filter-preview');
      if (!previewEl) return;
      if (!pendingPhotoCanvas) {
        previewEl.innerHTML = '';
        document.getElementById('btn-bulk-photo-confirm').disabled = true;
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
          renderBulkPhotoFilterPreview();
        });
      });
      document.getElementById('btn-bulk-photo-confirm').disabled = false;
    }

    document.getElementById('bulk-photo-file-input').addEventListener('change', async (e) => {
      const file = e.target.files[0];
      if (!file) return;
      pendingPhotoCanvas = await resizeImageToCanvas(file, CONFIG.PHOTO_MAX_EDGE);
      pendingPhotoFilter = 'original';
      renderBulkPhotoFilterPreview();
    });

    document.getElementById('btn-bulk-photo-confirm').addEventListener('click', async () => {
      if (!pendingPhotoCanvas) return;
      const hasExistingPhoto = ev.members.some(m => !!photoStore.get(eventId, m.id));
      if (hasExistingPhoto) {
        const ok = await confirmDialog('기존에 첨부된 사진도 모두 새 사진으로 덮어씁니다. 계속할까요?');
        if (!ok) return;
      }
      const finalCanvas = applyPhotoFilter(pendingPhotoCanvas, pendingPhotoFilter);
      const dataUrl = canvasToCompressedDataURL(finalCanvas, CONFIG.PHOTO_QUALITY);
      let appliedCount = 0;
      try {
        for (const m of ev.members) {
          photoStore.set(eventId, m.id, dataUrl);
          appliedCount++;
        }
      } catch (err) {
        if (err.name === 'QuotaExceededError') {
          closeModal();
          showToast(`${appliedCount}/${ev.members.length}명에게 적용한 뒤 용량이 가득 찼어요`);
          openPhotoManagerSheet(eventId);
          return;
        }
        throw err;
      }
      closeModal();
      showToast(`전체 ${appliedCount}명에게 사진을 적용했어요`);
      openPhotoManagerSheet(eventId);
    });
  }

  async function captureReceiptToBlob(receiptEl) {
```

- [ ] **Step 4: 브라우저에서 수동 검증**

`index.html`을 브라우저로 열고 다음을 확인한다:
1. 멤버가 2명 이상인 이벤트를 만들고(또는 기존 이벤트 사용) 영수증 화면 → "🖼 사진 관리" 시트를 연다. "전체 멤버에게 같은 사진 적용" 버튼이 보인다.
2. 버튼 클릭 → 파일 선택 → 필터(원본/흑백/도트) 프리뷰가 뜨고 하나를 고를 수 있다 → "전체 적용" 클릭.
3. 아직 아무도 사진이 없는 상태라면 확인창 없이 바로 적용되고, "전체 N명에게 사진을 적용했어요" 토스트가 뜬다. 사진 관리 시트를 다시 열면 모든 멤버 칸에 같은 사진이 보인다.
4. 이번엔 특정 멤버 하나에 개별 사진을 먼저 첨부한 뒤 다시 일괄 적용을 실행 — "기존에 첨부된 사진도 모두 새 사진으로 덮어씁니다. 계속할까요?" 확인창이 뜨는지 확인하고, 확인을 누르면 그 멤버 사진도 새 사진으로 교체되는지 확인한다.
5. 각 멤버 탭으로 이동해 영수증에 사진이 실제로 반영되는지 확인한다.

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
사진 관리 시트에 전체 멤버 일괄 사진 적용 기능 추가

EOF
)"
```
