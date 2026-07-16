# Stage 2 (결제 항목 입력 · 정산 엔진 · 결과 표) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add item entry (`event.items[]`), a pure settlement-calculation engine (`calcSettlement`), and a settlement result screen to the existing single-file `index.html`, on top of the Stage 1 skeleton (event/member CRUD, hash routing).

**Architecture:** Everything lives in the same `<script>` block in `index.html`. The settlement engine is a pure function with zero DOM/store access — screens call it and render its output. Two new routes (`#/event/:id/items`, `#/event/:id/result`) are added using the existing hardcoded-regex router style (no generic pattern matcher). Auto-save on the new items screen reuses the existing `.save-indicator` visual language, but is triggered per discrete action (add/edit/delete/reorder) rather than per-keystroke debounce, since item entry is a structured multi-field form, not free-text fields like Stage 1's settings screen — see Task 4 for details.

**Tech Stack:** Vanilla JS/HTML/CSS (no frameworks), same as Stage 1. No new CDN dependencies.

## Global Constraints

- Single `index.html` file. No build tools, npm, bundler, or framework.
- No new CDN dependencies this stage.
- All new tunable values go in `CONFIG`: `ROUNDING`, `ITEM_EMOJIS`, `CURRENCY_LOCALE`, `DEV_MODE`.
- Existing CONFIG-color scoping decision from Stage 1 stands: only `COLOR_PRIMARY`/`COLOR_BG`/`MEMBER_COLORS` live in CONFIG and get injected as CSS custom properties. New literal CSS colors (e.g. the settlement table's total-row highlight) are NOT added to CONFIG.
- Comments in Korean.
- Mobile-first; `#app` stays `max-width: 560px` centered (unchanged from Stage 1).
- `event.items[]` field names are fixed for future Firestore migration — see spec §2. Never store amounts as floats.
- `calcSettlement(event)` is a pure function: no `store`, no DOM access, no side effects. Screens call it and render the result.
- Settlement results are never persisted — always recomputed from `event.items` on render.
- Integrity violations (`errors[]`) never block saving an item; they only block transitioning `status` to `'settled'`.
- `runRegressionTests()` auto-runs on `DOMContentLoaded` only when `CONFIG.DEV_MODE === true`, mirroring Stage 1's `runSelfTests()` console pattern.
- Reference spec: `docs/superpowers/specs/2026-07-17-stage2-settlement-design.md`.

## File Structure

Single file, `index.html` (currently 743 lines after Stage 1). All tasks modify this one file — there is no separate test file or module system in this codebase. Because tasks land as sequential commits, insertion points are described by **anchor text** (existing comments/function names to search for) rather than fixed line numbers, since line numbers shift as earlier tasks land.

New code sections added, in file order:
1. `CONFIG` — 4 new fields (Task 1)
2. New "정산 관련 도메인 함수" section, placed right after `setOwner` and before `// ====== 자체 테스트 ======` (Tasks 1–2): `addItem`, `updateItem`, `removeItem`, `moveItemOrder`, `calcSettlement`, `distributeFloor1`, `distributeRemainder`
3. New `runRegressionTests()`, placed right after `runSelfTests()` (Tasks 1–2)
4. `matchRoute` extended (Task 3)
5. `renderEventCard` modified, `#btn-goto-items` handler modified (Task 3)
6. New "화면: 결제 입력" section — `renderItemsEntry`, `renderItemForm`, `renderItemCard`, `formatAmountInput`, `parseAmountInput`, `readFixedAmounts`, `updateFixedPreview` (Task 4)
7. New "화면: 정산 결과" section — `renderSettlementResult` (Task 5)
8. Route registration + `DEV_MODE` auto-run wiring, at the bottom near existing `registerRoute`/`DOMContentLoaded` calls (Tasks 4–5)
9. New CSS blocks inside the existing `<style>` block, each under its own Korean section comment (Tasks 1, 4, 5, 6)
10. `CLAUDE.md` updates (Task 6)

---

### Task 1: CONFIG 확장 + 항목 도메인 함수 + calcSettlement(floor1) + 회귀 테스트 뼈대

**Files:**
- Modify: `index.html` (CONFIG object; new domain-functions section after `setOwner`; new `runRegressionTests()` after `runSelfTests()`)

**Interfaces:**
- Consumes: `CONFIG`, `createEvent`, `addMember`, `setOwner` (Stage 1, unchanged)
- Produces: `CONFIG.ROUNDING`/`ITEM_EMOJIS`/`CURRENCY_LOCALE`/`DEV_MODE`; `addItem(event, data)`, `updateItem(event, itemId, data)`, `removeItem(event, itemId)`, `moveItemOrder(event, itemId, direction)`; `calcSettlement(event) → {matrix, itemTotals, memberTotals, grandTotal, transfers, errors}`; `distributeFloor1(event, participantIds, remainingAmount, resultMap, bonusMemberId)`; `distributeRemainder(event, item, participantIds, remainingAmount, resultMap)` (floor1-only in this task — Task 2 replaces its body); `runRegressionTests()`

- [ ] **Step 1: Extend CONFIG**

Find the `CONFIG` object (search for `COLOR_BG: '#FFF9F2',`) and add four fields immediately before the closing `};`:

```javascript
    COLOR_PRIMARY: '#FF7A45',
    COLOR_BG: '#FFF9F2',
    ROUNDING: 'floor1',            // 'floor1' | 'ceil10' | 'ceil100'
    ITEM_EMOJIS: ['🍽️','☕','🍦','📷','🚗','🎮','🎫','🛍️','🍺','🧾'],
    CURRENCY_LOCALE: 'ko-KR',
    DEV_MODE: false,
  };
```

- [ ] **Step 2: Add item CRUD domain functions**

Find the `setOwner` function (search for `function setOwner(event, memberId) {`). Immediately after its closing `}`, insert:

```javascript

  // ====== 도메인 함수: 결제 항목 ======
  function addItem(event, data) {
    const trimmed = (data.name || '').trim();
    if (!trimmed) throw new Error('항목명을 입력해주세요');
    if (!data.participantIds || data.participantIds.length === 0) {
      throw new Error('참여자를 1명 이상 선택해주세요');
    }
    const fixedAmounts = data.fixedAmounts || {};
    const fixedSum = Object.values(fixedAmounts).reduce((s, v) => s + v, 0);
    if (fixedSum > data.amount) {
      throw new Error('고정 금액 합이 총액을 초과합니다');
    }
    const item = {
      id: crypto.randomUUID(),
      name: trimmed,
      emoji: data.emoji || '',
      amount: data.amount,
      participantIds: data.participantIds.slice(),
      fixedAmounts: Object.assign({}, fixedAmounts),
      memo: data.memo || '',
      order: event.items.length,
    };
    event.items.push(item);
    if (event.status === 'settled') event.status = 'draft';
    return item;
  }

  function updateItem(event, itemId, data) {
    const item = event.items.find(i => i.id === itemId);
    if (!item) throw new Error('항목을 찾을 수 없습니다');
    const trimmed = (data.name || '').trim();
    if (!trimmed) throw new Error('항목명을 입력해주세요');
    if (!data.participantIds || data.participantIds.length === 0) {
      throw new Error('참여자를 1명 이상 선택해주세요');
    }
    const fixedAmounts = data.fixedAmounts || {};
    const fixedSum = Object.values(fixedAmounts).reduce((s, v) => s + v, 0);
    if (fixedSum > data.amount) {
      throw new Error('고정 금액 합이 총액을 초과합니다');
    }
    item.name = trimmed;
    item.emoji = data.emoji || '';
    item.amount = data.amount;
    item.participantIds = data.participantIds.slice();
    item.fixedAmounts = Object.assign({}, fixedAmounts);
    item.memo = data.memo || '';
    if (event.status === 'settled') event.status = 'draft';
    return item;
  }

  function removeItem(event, itemId) {
    const idx = event.items.findIndex(i => i.id === itemId);
    if (idx === -1) return;
    event.items.splice(idx, 1);
    event.items.forEach((item, i) => { item.order = i; });
    if (event.status === 'settled') event.status = 'draft';
  }

  function moveItemOrder(event, itemId, direction) {
    const sorted = event.items.slice().sort((a, b) => a.order - b.order);
    const idx = sorted.findIndex(i => i.id === itemId);
    if (idx === -1) return;
    const swapIdx = direction === 'up' ? idx - 1 : idx + 1;
    if (swapIdx < 0 || swapIdx >= sorted.length) return;
    const tmp = sorted[idx].order;
    sorted[idx].order = sorted[swapIdx].order;
    sorted[swapIdx].order = tmp;
  }
```

`addItem`/`updateItem`/`removeItem` reverting `status` from `'settled'` to `'draft'` implements the spec's "항목 수정 시 자동으로 draft 복귀" requirement directly in the domain layer (cleaner than duplicating this in every UI call site).

- [ ] **Step 3: Add the settlement engine**

Immediately after the item-CRUD functions from Step 2, insert:

```javascript

  // ====== 정산 계산 엔진 (순수 함수 — DOM/store 접근 금지) ======
  function calcSettlement(event) {
    const matrix = {};
    const itemTotals = {};
    const memberTotals = {};
    const errors = [];
    event.members.forEach(m => { memberTotals[m.id] = 0; });

    event.items.forEach(item => {
      matrix[item.id] = {};
      itemTotals[item.id] = item.amount;

      if (item.participantIds.length === 0) {
        errors.push({ itemId: item.id, code: 'NO_PARTICIPANT', message: `"${item.name}": 참여자가 없습니다` });
        return;
      }

      const fixedAmounts = item.fixedAmounts || {};
      const fixedKeys = Object.keys(fixedAmounts);
      const invalidFixedKey = fixedKeys.find(id => !item.participantIds.includes(id));
      if (invalidFixedKey) {
        errors.push({ itemId: item.id, code: 'FIXED_NOT_PARTICIPANT', message: `"${item.name}": 고정 금액 대상이 참여자 목록에 없습니다` });
        return;
      }

      const fixedSum = fixedKeys.reduce((sum, id) => sum + fixedAmounts[id], 0);
      if (fixedSum > item.amount) {
        errors.push({ itemId: item.id, code: 'FIXED_EXCEEDS', message: `"${item.name}": 고정 금액 합(₩${fixedSum})이 총액(₩${item.amount})을 초과합니다` });
        return;
      }

      fixedKeys.forEach(id => { matrix[item.id][id] = fixedAmounts[id]; });

      const remainderParticipants = item.participantIds.filter(id => !fixedKeys.includes(id));
      const remainingAmount = item.amount - fixedSum;
      if (remainderParticipants.length > 0) {
        distributeRemainder(event, item, remainderParticipants, remainingAmount, matrix[item.id]);
      }

      const distributedSum = item.participantIds.reduce((sum, id) => sum + (matrix[item.id][id] || 0), 0);
      if (distributedSum !== item.amount) {
        errors.push({ itemId: item.id, code: 'INTEGRITY_MISMATCH', message: `"${item.name}": 분배액 합(₩${distributedSum})이 총액(₩${item.amount})과 일치하지 않습니다` });
      }

      item.participantIds.forEach(id => {
        memberTotals[id] = (memberTotals[id] || 0) + (matrix[item.id][id] || 0);
      });
    });

    const grandTotal = event.items.reduce((sum, item) => sum + item.amount, 0);
    const owner = event.members.find(m => m.isOwner);
    const transfers = {};
    event.members.forEach(m => {
      if (!owner || m.id !== owner.id) {
        transfers[m.id] = memberTotals[m.id] || 0;
      }
    });

    return { matrix, itemTotals, memberTotals, grandTotal, transfers, errors };
  }

  // 잔여 금액을 floor1 규칙으로 분배: 몫은 내림, 끝전은 bonusMemberId(없으면 배열 순서상 첫 참여자)에게 가산
  function distributeFloor1(event, participantIds, remainingAmount, resultMap, bonusMemberId) {
    const count = participantIds.length;
    const share = Math.floor(remainingAmount / count);
    const remainder = remainingAmount - share * count;
    participantIds.forEach(id => { resultMap[id] = share; });
    const bonusId = bonusMemberId || event.members.find(m => participantIds.includes(m.id)).id;
    resultMap[bonusId] = (resultMap[bonusId] || 0) + remainder;
  }

  // CONFIG.ROUNDING에 따라 잔여 금액을 분배한다. 이 태스크에서는 floor1만 구현 — ceil10/ceil100은 Task 2에서 추가.
  function distributeRemainder(event, item, participantIds, remainingAmount, resultMap) {
    const owner = event.members.find(m => m.isOwner);
    const ownerParticipates = !!owner && participantIds.includes(owner.id);
    distributeFloor1(event, participantIds, remainingAmount, resultMap, ownerParticipates ? owner.id : null);
  }
```

- [ ] **Step 4: Add the regression test with seed A/B**

Find `runSelfTests()` (search for `function runSelfTests() {`) and its closing `}`. Immediately after that closing `}`, insert:

```javascript

  // ====== 회귀 테스트 (콘솔에서 runRegressionTests() 실행, CONFIG.DEV_MODE=true면 자동 실행) ======
  function runRegressionTests() {
    const results = [];
    function assert(desc, cond) {
      results.push({ desc, pass: !!cond });
    }

    function makeEventWithMembers(name, ownerName) {
      const ev = createEvent(name, '2026-01-01');
      ['인우', '찬우', '정표', '정우', '은찬'].forEach(n => addMember(ev, n));
      setOwner(ev, ev.members.find(m => m.name === ownerName).id);
      return ev;
    }

    // ---- 시드 A: 인당 39,820 / 총계 199,100 ----
    const evA = makeEventWithMembers('시드 A', '정우');
    const idsA = evA.members.map(m => m.id);
    [
      ['인우메뉴', 27000], ['찬우메뉴', 21500], ['정표메뉴', 28000], ['정우메뉴', 31500], ['은찬메뉴', 20900],
      ['오프닝 카페메뉴', 25500], ['포토이즘', 15000], ['라라스위토', 12000], ['주차비', 2000], ['더벤티', 15700],
    ].forEach(([n, amount]) => addItem(evA, { name: n, amount, participantIds: idsA }));
    const resultA = calcSettlement(evA);
    assert('시드 A: 인당 39,820원', idsA.every(id => resultA.memberTotals[id] === 39820));
    assert('시드 A: 총계 199,100원', resultA.grandTotal === 199100);
    assert('시드 A: 오류 없음', resultA.errors.length === 0);

    // ---- 시드 B: 인당 54,800 / 총계 274,000 ----
    const evB = makeEventWithMembers('시드 B', '정우');
    const idsB = evB.members.map(m => m.id);
    [
      ['돈까스', 45500], ['모앙떡', 27000], ['옹근달 카페', 32500],
      ['방탈출', 75000], ['포토랩 라이뷰러리', 44000], ['방탈출 예약금', 50000],
    ].forEach(([n, amount]) => addItem(evB, { name: n, amount, participantIds: idsB }));
    const resultB = calcSettlement(evB);
    assert('시드 B: 인당 54,800원', idsB.every(id => resultB.memberTotals[id] === 54800));
    assert('시드 B: 총계 274,000원', resultB.grandTotal === 274000);
    assert('시드 B: 오류 없음', resultB.errors.length === 0);

    // ---- 단위 테스트: 끝전 (10,000원 ÷ 3인, 총무 포함) ----
    const evC = createEvent('끝전 테스트', '2026-01-01');
    addMember(evC, '총무'); // 첫 멤버 = 자동으로 총무
    addMember(evC, 'B');
    addMember(evC, 'C');
    addItem(evC, { name: '테스트항목', amount: 10000, participantIds: evC.members.map(m => m.id) });
    const resultC = calcSettlement(evC);
    const ownerC = evC.members.find(m => m.name === '총무').id;
    const bC = evC.members.find(m => m.name === 'B').id;
    const cC = evC.members.find(m => m.name === 'C').id;
    assert('끝전: 총무 3,334원', resultC.memberTotals[ownerC] === 3334);
    assert('끝전: B 3,333원', resultC.memberTotals[bC] === 3333);
    assert('끝전: C 3,333원', resultC.memberTotals[cC] === 3333);

    // ---- 단위 테스트: 참여자 제외 (시드 B 방탈출 75,000원에서 은찬 제외) ----
    const evD = makeEventWithMembers('참여자 제외 테스트', '정우');
    const eunchanD = evD.members.find(m => m.name === '은찬').id;
    const idsWithoutEunchan = evD.members.filter(m => m.id !== eunchanD).map(m => m.id);
    addItem(evD, { name: '방탈출', amount: 75000, participantIds: idsWithoutEunchan });
    const resultD = calcSettlement(evD);
    const itemD = evD.items[0].id;
    assert('참여자 제외: 4인 각 18,750원', idsWithoutEunchan.every(id => resultD.matrix[itemD][id] === 18750));
    assert('참여자 제외: 은찬 키 없음', resultD.matrix[itemD][eunchanD] === undefined);

    // ---- 단위 테스트: 고정 금액 (더벤티 15,700원에서 정표 고정 3,640원) ----
    const evE = makeEventWithMembers('고정 금액 테스트', '정우');
    const jeongpyoE = evE.members.find(m => m.name === '정표').id;
    addItem(evE, { name: '더벤티', amount: 15700, participantIds: evE.members.map(m => m.id), fixedAmounts: { [jeongpyoE]: 3640 } });
    const resultE = calcSettlement(evE);
    const itemE = evE.items[0].id;
    const othersE = evE.members.filter(m => m.id !== jeongpyoE).map(m => m.id);
    assert('고정 금액: 정표 3,640원', resultE.matrix[itemE][jeongpyoE] === 3640);
    assert('고정 금액: 나머지 4인 각 3,015원', othersE.every(id => resultE.matrix[itemE][id] === 3015));

    // ---- 단위 테스트: 에러 검출 (고정 합 초과) — addItem은 이 상태를 막으므로 calcSettlement 방어 로직만 직접 검증 ----
    const evF = createEvent('에러 테스트', '2026-01-01');
    addMember(evF, 'A');
    addMember(evF, 'B');
    const idsF = evF.members.map(m => m.id);
    evF.items.push({ id: crypto.randomUUID(), name: '초과 테스트', emoji: '', amount: 1000, participantIds: idsF, fixedAmounts: { [idsF[0]]: 2000 }, memo: '', order: 0 });
    const resultF = calcSettlement(evF);
    assert('에러 검출: FIXED_EXCEEDS', resultF.errors.some(e => e.code === 'FIXED_EXCEEDS'));

    // ---- 전 테스트 공통: Σ분배액 === amount ----
    [evA, evB, evC, evD, evE].forEach(ev => {
      const r = calcSettlement(ev);
      ev.items.forEach(item => {
        const sum = Object.values(r.matrix[item.id] || {}).reduce((s, v) => s + v, 0);
        assert(`${ev.name} - "${item.name}": 분배액 합 === amount`, sum === item.amount);
      });
    });

    console.table(results);
    const failed = results.filter(r => !r.pass);
    console.log(failed.length === 0 ? '✅ 회귀 테스트 전체 통과' : `❌ ${failed.length}건 실패`);
    return results;
  }
```

- [ ] **Step 5: Run the regression test in a real browser and verify it passes**

This codebase has no test runner — verification happens in the browser console, same as Stage 1's `runSelfTests()`.

Run: `python -m http.server 8123` from the project root, then (Chrome automation blocks `file://` — always use the local server) navigate to `http://localhost:8123/index.html` and evaluate in the page console:

```javascript
JSON.stringify(runRegressionTests())
```

Expected: every entry has `"pass":true`. If `시드 A: 인당 39,820원` or `시드 B: 인당 54,800원` fail, the bug is in `calcSettlement`/`distributeFloor1` — do not adjust the expected seed values, they are the CLAUDE.md-mandated regression fixture.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Stage 2: CONFIG 확장 + 항목 도메인 함수 + calcSettlement(floor1) + 회귀 테스트"
```

---

### Task 2: ceil10 / ceil100 라운딩 정책 + 폴백 + 단위 테스트

**Files:**
- Modify: `index.html` (`distributeRemainder` function body; `runRegressionTests()`)

**Interfaces:**
- Consumes: `CONFIG.ROUNDING`, `distributeFloor1` (Task 1)
- Produces: updated `distributeRemainder` supporting `'ceil10'`/`'ceil100'` with negative-owner-share fallback to `floor1`

- [ ] **Step 1: Replace `distributeRemainder`'s body**

Find the `distributeRemainder` function added in Task 1 and replace its entire body:

```javascript
  // CONFIG.ROUNDING에 따라 잔여 금액을 분배한다.
  function distributeRemainder(event, item, participantIds, remainingAmount, resultMap) {
    const owner = event.members.find(m => m.isOwner);
    const ownerParticipates = !!owner && participantIds.includes(owner.id);

    if (CONFIG.ROUNDING === 'floor1' || !ownerParticipates) {
      distributeFloor1(event, participantIds, remainingAmount, resultMap, ownerParticipates ? owner.id : null);
      return;
    }

    const unit = CONFIG.ROUNDING === 'ceil100' ? 100 : 10;
    const others = participantIds.filter(id => id !== owner.id);
    const equalShare = remainingAmount / participantIds.length;
    const roundedShare = Math.ceil(equalShare / unit) * unit;
    const othersSum = roundedShare * others.length;
    const ownerShare = remainingAmount - othersSum;

    if (ownerShare < 0) {
      console.warn(`항목 "${item.name}"(id: ${item.id}): ${CONFIG.ROUNDING} 라운딩 시 총무 몫이 음수(${ownerShare})가 되어 floor1로 폴백합니다`);
      distributeFloor1(event, participantIds, remainingAmount, resultMap, owner.id);
      return;
    }

    others.forEach(id => { resultMap[id] = roundedShare; });
    resultMap[owner.id] = ownerShare;
  }
```

- [ ] **Step 2: Add ceil10/ceil100 + fallback unit tests**

Find the "전 테스트 공통: Σ분배액 === amount" block inside `runRegressionTests()` (added in Task 1) and insert the following immediately **before** it (these tests temporarily swap `CONFIG.ROUNDING` and restore it afterward, so they don't affect the floor1-based seed A/B assertions above them):

```javascript

    // ---- 단위 테스트: ceil10 라운딩 ----
    const originalRounding = CONFIG.ROUNDING;
    CONFIG.ROUNDING = 'ceil10';
    const evG = createEvent('ceil10 테스트', '2026-01-01');
    addMember(evG, '총무'); addMember(evG, 'B'); addMember(evG, 'C');
    addItem(evG, { name: 'ceil10 항목', amount: 1000, participantIds: evG.members.map(m => m.id) });
    const resultG = calcSettlement(evG);
    const ownerG = evG.members.find(m => m.name === '총무').id;
    const bG = evG.members.find(m => m.name === 'B').id;
    const cG = evG.members.find(m => m.name === 'C').id;
    const itemG = evG.items[0].id;
    assert('ceil10: B/C 각 340원', resultG.matrix[itemG][bG] === 340 && resultG.matrix[itemG][cG] === 340);
    assert('ceil10: 총무 320원', resultG.matrix[itemG][ownerG] === 320);

    // ---- 단위 테스트: ceil100 라운딩 ----
    CONFIG.ROUNDING = 'ceil100';
    const evH = createEvent('ceil100 테스트', '2026-01-01');
    addMember(evH, '총무'); addMember(evH, 'B'); addMember(evH, 'C');
    addItem(evH, { name: 'ceil100 항목', amount: 1000, participantIds: evH.members.map(m => m.id) });
    const resultH = calcSettlement(evH);
    const ownerH = evH.members.find(m => m.name === '총무').id;
    const bH = evH.members.find(m => m.name === 'B').id;
    const cH = evH.members.find(m => m.name === 'C').id;
    const itemH = evH.items[0].id;
    assert('ceil100: B/C 각 400원', resultH.matrix[itemH][bH] === 400 && resultH.matrix[itemH][cH] === 400);
    assert('ceil100: 총무 200원', resultH.matrix[itemH][ownerH] === 200);

    // ---- 단위 테스트: ceil100 음수 폴백 → floor1 ----
    const evI = createEvent('ceil100 폴백 테스트', '2026-01-01');
    addMember(evI, '총무'); addMember(evI, 'B'); addMember(evI, 'C');
    addItem(evI, { name: 'ceil100 폴백 항목', amount: 100, participantIds: evI.members.map(m => m.id) });
    const resultI = calcSettlement(evI);
    const ownerI = evI.members.find(m => m.name === '총무').id;
    const bI = evI.members.find(m => m.name === 'B').id;
    const cI = evI.members.find(m => m.name === 'C').id;
    const itemI = evI.items[0].id;
    assert('ceil100 폴백: 총무 34원(floor1로 폴백)', resultI.matrix[itemI][ownerI] === 34);
    assert('ceil100 폴백: B/C 각 33원', resultI.matrix[itemI][bI] === 33 && resultI.matrix[itemI][cI] === 33);
    CONFIG.ROUNDING = originalRounding;
```

- [ ] **Step 3: Run in browser and verify all pass**

Same as Task 1 Step 5: `JSON.stringify(runRegressionTests())` in the console at `http://localhost:8123/index.html`, confirm every entry `"pass":true`, including the 5 new ceil10/ceil100/fallback assertions and that the seed A/B assertions above them are unaffected (since `CONFIG.ROUNDING` is restored via `originalRounding` right after the ceil tests run).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Stage 2: ceil10/ceil100 라운딩 정책 + 음수 폴백 + 단위 테스트 추가"
```

---

### Task 3: 라우터 확장 + 결제 입력 버튼 연결 + 홈 카드 총액 반영

**Files:**
- Modify: `index.html` (`matchRoute`; `#btn-goto-items` handler + its HTML class in `renderEventSettings`; `renderEventCard`)

**Interfaces:**
- Consumes: `calcSettlement` (Task 1), existing `matchRoute`/`registerRoute`/`renderEventCard`/`renderEventSettings` (Stage 1)
- Produces: `matchRoute` recognizing `/event/:id/items` and `/event/:id/result` hash patterns (routes themselves are registered in Tasks 4 and 5, once their render functions exist — registering a route before its render function is defined would throw a `ReferenceError` and break the whole app, so this task only teaches `matchRoute` to recognize the URL shape); `#btn-goto-items` navigates instead of showing a toast; home card shows real `grandTotal`

- [ ] **Step 1: Extend `matchRoute`**

Find `matchRoute` (search for `function matchRoute(hash) {`). It currently ends with:

```javascript
    const eventMatch = hash.match(/^\/event\/([^/]+)$/);
    if (eventMatch && routes['/event/:id']) {
      return { render: routes['/event/:id'], params: { id: eventMatch[1] } };
    }
    return null;
  }
```

Replace it with:

```javascript
    const eventMatch = hash.match(/^\/event\/([^/]+)$/);
    if (eventMatch && routes['/event/:id']) {
      return { render: routes['/event/:id'], params: { id: eventMatch[1] } };
    }
    const itemsMatch = hash.match(/^\/event\/([^/]+)\/items$/);
    if (itemsMatch && routes['/event/:id/items']) {
      return { render: routes['/event/:id/items'], params: { id: itemsMatch[1] } };
    }
    const resultMatch = hash.match(/^\/event\/([^/]+)\/result$/);
    if (resultMatch && routes['/event/:id/result']) {
      return { render: routes['/event/:id/result'], params: { id: resultMatch[1] } };
    }
    return null;
  }
```

- [ ] **Step 2: Activate the "결제 입력으로 →" button**

Find, in `renderEventSettings`, the button markup:

```html
        <button type="button" id="btn-goto-items" class="btn btn-disabled">결제 입력으로 →</button>
```

Replace with:

```html
        <button type="button" id="btn-goto-items" class="btn btn-primary">결제 입력으로 →</button>
```

Find its click handler:

```javascript
    document.getElementById('btn-goto-items').addEventListener('click', () => {
      showToast('Stage 2에서 열립니다');
    });
```

Replace with:

```javascript
    document.getElementById('btn-goto-items').addEventListener('click', () => {
      location.hash = `#/event/${ev.id}/items`;
    });
```

- [ ] **Step 3: Show the real total on the home screen**

Find `renderEventCard` (search for `function renderEventCard(ev) {`). Replace it entirely:

```javascript
  function renderEventCard(ev) {
    const emojis = ev.members.map(m => m.emoji).join(' ');
    const grandTotal = calcSettlement(ev).grandTotal;
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
        <div class="card-total">₩${grandTotal.toLocaleString(CONFIG.CURRENCY_LOCALE)}</div>
      </div>
    `;
  }
```

- [ ] **Step 4: Verify manually in the browser**

At `http://localhost:8123/index.html`: create an event, add a member, click "결제 입력으로 →" — confirm the hash becomes `#/event/{id}/items` (it will bounce back to `#/` since no route is registered for it yet — that's expected until Task 4 lands; just confirm the hash briefly changes and no console error is thrown other than the expected redirect). Confirm the home card shows `₩0` for an event with no items (since `calcSettlement` on an empty `items` array returns `grandTotal: 0`).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Stage 2: 라우터에 items/result 패턴 추가, 결제 입력 버튼 연결, 홈 카드 총액 반영"
```

---

### Task 4: 결제 입력 화면 — 목록 · 추가/수정 폼 · 삭제 · 순서 변경 · 하단 요약바

**Files:**
- Modify: `index.html` (`<style>` block: new CSS section; `<script>` block: new "화면: 결제 입력" section; route registration at the bottom)

**Interfaces:**
- Consumes: `store`, `escapeHtml`, `escapeAttr`, `showToast`, `confirmDialog`, `renderBadge`, `CONFIG.ITEM_EMOJIS`/`CURRENCY_LOCALE`, `addItem`/`updateItem`/`removeItem`/`moveItemOrder`, `calcSettlement` (Tasks 1–2), `registerRoute` (Stage 1)
- Produces: `renderItemsEntry(params)` (registers as `/event/:id/items`), `renderItemForm(ev, editing)`, `renderItemCard(item, ev, index, total)`, `formatAmountInput(inputEl)`, `parseAmountInput(inputEl)`, `readFixedAmounts()`, `updateFixedPreview(ev)`, module-scoped `editingItemId`, `selectedItemParticipants`

**Design note on auto-save:** unlike Stage 1's settings screen (free-text fields that autosave on every keystroke via `flushSave`/`scheduleSave`), this screen's item data is only committed via an explicit "항목 추가"/"수정 완료" button click (multi-field validation — participant selection, fixed-amount caps — must happen atomically when the user finishes the form, not mid-keystroke). Every discrete mutation (add/update/delete/reorder) calls `store.save(ev)` immediately and flashes the same `.save-indicator` span with "저장됨 ✓", reusing Stage 1's visual save-confirmation language even though the trigger differs from a debounce timer.

- [ ] **Step 1: Add CSS for the items screen**

Find the `/* ====== 반응형 보강 ====== */` comment near the end of the `<style>` block. Insert a new section immediately **before** it:

```css
  /* ====== 결제 입력 ====== */
  .item-list { display: flex; flex-direction: column; gap: 8px; }
  .item-card {
    display: flex;
    flex-direction: column;
    gap: 6px;
    padding: 10px 12px;
    border-radius: 12px;
    background: #F9F4EC;
  }
  .item-card-top { display: flex; align-items: center; gap: 8px; }
  .item-emoji { font-size: 18px; }
  .item-name { flex: 1; font-weight: 600; }
  .item-amount { font-weight: 700; color: var(--color-primary); }
  .item-card-actions { display: flex; justify-content: flex-end; gap: 4px; align-items: center; }
  .item-order-btn {
    background: none; border: none; font-size: 12px; color: #9A8C7C;
    min-width: 32px; min-height: 32px;
  }
  .item-order-btn:disabled { opacity: 0.3; }
  .item-delete-btn {
    background: none; border: none; color: #6B5B4B; font-size: 14px;
    min-width: 32px; min-height: 32px;
  }
  .item-participants { display: flex; align-items: center; gap: 4px; font-size: 16px; flex-wrap: wrap; }
  .item-participant-inactive { opacity: 0.35; }

  .item-add-form { display: flex; flex-direction: column; gap: 8px; }
  .participant-toggle-list { display: flex; flex-wrap: wrap; gap: 6px; }
  .participant-toggle-chip {
    border: 2px solid transparent;
    border-radius: 999px;
    padding: 6px 12px;
    font-size: 13px;
    font-weight: 600;
    background: #F5EEE4;
    opacity: 0.4;
    min-height: 36px;
  }
  .participant-toggle-chip.active {
    border-color: var(--color-primary);
    opacity: 1;
  }
  .participant-toggle-actions { display: flex; gap: 8px; }
  .item-form-actions { display: flex; gap: 8px; }

  .item-summary-bar {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    background: #fff;
    box-shadow: 0 -2px 8px rgba(0,0,0,0.08);
    padding: 10px 16px;
    display: flex;
    align-items: center;
    gap: 12px;
    z-index: 500;
  }
  .item-summary-members {
    flex: 1;
    display: flex;
    flex-wrap: nowrap;
    gap: 10px;
    overflow-x: auto;
    font-size: 13px;
  }
  .item-summary-chip { white-space: nowrap; }

  #advanced-options summary {
    cursor: pointer;
    font-size: 13px;
    font-weight: 600;
    color: #6B5B4B;
  }
  .fixed-amount-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }
  .fixed-amount-row .input-fixed-amount { width: 120px; }

```

- [ ] **Step 2: Add the items screen functions**

Find the `// ====== 라우트 등록 & 초기 실행 ======` comment near the end of the `<script>` block. Insert the entire new section immediately **before** it:

```javascript
  // ====== 화면: 결제 입력 ======
  let editingItemId = null;
  let selectedItemParticipants = null;

  function renderItemsEntry(params, opts) {
    opts = opts || {};
    const app = document.getElementById('app');
    const ev = store.get(params.id);
    if (!ev) { location.hash = '#/'; return; }

    const sortedItems = ev.items.slice().sort((a, b) => a.order - b.order);
    const settlement = calcSettlement(ev);
    const editing = editingItemId ? ev.items.find(i => i.id === editingItemId) : null;

    app.innerHTML = `
      <header class="topbar">
        <button type="button" class="btn-back" id="btn-back">←</button>
        <h1>결제 입력</h1>
        <span id="save-indicator" class="save-indicator">${opts.justSaved ? '저장됨 ✓' : ''}</span>
      </header>
      <main class="screen">
        <section class="card-section">
          <h2>항목 (${ev.items.length})</h2>
          <div id="item-list" class="item-list">${sortedItems.map((item, i) => renderItemCard(item, ev, i, sortedItems.length)).join('')}</div>
          ${ev.items.length === 0 ? '<div class="field-hint">결제 항목을 추가해주세요</div>' : ''}
        </section>

        <section class="card-section">
          <h2>${editing ? '항목 수정' : '항목 추가'}</h2>
          ${renderItemForm(ev, editing)}
        </section>
      </main>
      <div class="item-summary-bar">
        <div class="item-summary-members">
          ${ev.members.map(m => `<span class="item-summary-chip">${m.emoji} ₩${(settlement.memberTotals[m.id] || 0).toLocaleString(CONFIG.CURRENCY_LOCALE)}</span>`).join('')}
        </div>
        <button type="button" id="btn-goto-result" class="btn btn-primary">정산 결과 보기 →</button>
      </div>
    `;

    document.getElementById('btn-back').addEventListener('click', () => { location.hash = `#/event/${ev.id}`; });
    document.getElementById('btn-goto-result').addEventListener('click', () => { location.hash = `#/event/${ev.id}/result`; });

    const initialParticipantIds = editing ? editing.participantIds : ev.members.map(m => m.id);
    selectedItemParticipants = new Set(initialParticipantIds);
    let selectedItemEmoji = editing ? editing.emoji : CONFIG.ITEM_EMOJIS[0];

    document.querySelectorAll('#item-emoji-picker .emoji-btn').forEach(btn => {
      btn.addEventListener('click', () => {
        document.querySelectorAll('#item-emoji-picker .emoji-btn').forEach(b => b.classList.remove('selected'));
        btn.classList.add('selected');
        selectedItemEmoji = btn.dataset.emoji;
        document.getElementById('input-item-emoji-custom').value = '';
      });
    });

    document.getElementById('input-item-amount').addEventListener('input', (e) => {
      formatAmountInput(e.target);
      updateFixedPreview(ev);
    });

    document.querySelectorAll('.participant-toggle-chip').forEach(chip => {
      chip.addEventListener('click', () => {
        const id = chip.dataset.id;
        if (selectedItemParticipants.has(id)) {
          selectedItemParticipants.delete(id);
          chip.classList.remove('active');
        } else {
          selectedItemParticipants.add(id);
          chip.classList.add('active');
        }
        updateFixedPreview(ev);
      });
    });

    document.getElementById('btn-select-all').addEventListener('click', () => {
      document.querySelectorAll('.participant-toggle-chip').forEach(chip => {
        selectedItemParticipants.add(chip.dataset.id);
        chip.classList.add('active');
      });
      updateFixedPreview(ev);
    });

    document.getElementById('btn-deselect-all').addEventListener('click', () => {
      document.querySelectorAll('.participant-toggle-chip').forEach(chip => {
        selectedItemParticipants.delete(chip.dataset.id);
        chip.classList.remove('active');
      });
      updateFixedPreview(ev);
    });

    document.querySelectorAll('.input-fixed-amount').forEach(input => {
      input.addEventListener('input', (e) => {
        formatAmountInput(e.target);
        updateFixedPreview(ev);
      });
    });

    updateFixedPreview(ev);

    document.getElementById('btn-save-item').addEventListener('click', () => {
      const errorEl = document.getElementById('item-form-error');
      errorEl.textContent = '';
      const name = document.getElementById('input-item-name').value;
      const customEmoji = document.getElementById('input-item-emoji-custom').value.trim();
      const emoji = customEmoji || selectedItemEmoji || '';
      const amount = parseAmountInput(document.getElementById('input-item-amount'));
      const memo = document.getElementById('input-item-memo').value;
      const participantIds = Array.from(selectedItemParticipants);
      const fixedAmounts = readFixedAmounts();
      const fixedSum = Object.values(fixedAmounts).reduce((s, v) => s + v, 0);
      if (fixedSum > amount) {
        errorEl.textContent = '고정 금액 합이 총액을 초과합니다';
        return;
      }
      try {
        if (editingItemId) {
          updateItem(ev, editingItemId, { name, emoji, amount, participantIds, fixedAmounts, memo });
        } else {
          addItem(ev, { name, emoji, amount, participantIds, fixedAmounts, memo });
        }
        store.save(ev);
        editingItemId = null;
        renderItemsEntry(params, { justSaved: true });
      } catch (err) {
        errorEl.textContent = err.message;
      }
    });

    if (editing) {
      document.getElementById('btn-cancel-edit').addEventListener('click', () => {
        editingItemId = null;
        renderItemsEntry(params);
      });
    }

    document.querySelectorAll('.item-edit-btn').forEach(btn => {
      btn.addEventListener('click', () => {
        editingItemId = btn.dataset.id;
        renderItemsEntry(params);
      });
    });

    document.querySelectorAll('.item-delete-btn').forEach(btn => {
      btn.addEventListener('click', async () => {
        const item = ev.items.find(i => i.id === btn.dataset.id);
        const ok = await confirmDialog(`"${item.name}" 항목을 삭제할까요?`);
        if (ok) {
          if (editingItemId === item.id) editingItemId = null;
          removeItem(ev, btn.dataset.id);
          store.save(ev);
          renderItemsEntry(params, { justSaved: true });
        }
      });
    });

    document.querySelectorAll('.item-order-btn').forEach(btn => {
      btn.addEventListener('click', () => {
        moveItemOrder(ev, btn.dataset.id, btn.dataset.direction);
        store.save(ev);
        renderItemsEntry(params, { justSaved: true });
      });
    });
  }

  function renderItemForm(ev, editing) {
    const amount = editing ? editing.amount : 0;
    const participantIds = editing ? editing.participantIds : ev.members.map(m => m.id);
    const fixedAmounts = editing ? editing.fixedAmounts : {};
    const isPresetEmoji = editing && CONFIG.ITEM_EMOJIS.includes(editing.emoji);
    return `
      <div class="item-add-form">
        <input id="input-item-name" class="input" type="text" placeholder="항목명" value="${editing ? escapeAttr(editing.name) : ''}" />
        <div class="emoji-picker" id="item-emoji-picker">
          ${CONFIG.ITEM_EMOJIS.map((e, i) => `<button type="button" class="emoji-btn${(editing ? editing.emoji === e : i === 0) ? ' selected' : ''}" data-emoji="${e}">${e}</button>`).join('')}
        </div>
        <input id="input-item-emoji-custom" class="input" type="text" maxlength="4" placeholder="이모지 직접 입력 (선택)" value="${editing && !isPresetEmoji ? escapeAttr(editing.emoji) : ''}" />
        <input id="input-item-amount" class="input" type="text" inputmode="numeric" placeholder="금액" value="${amount ? amount.toLocaleString(CONFIG.CURRENCY_LOCALE) : ''}" />
        <div class="field-label">참여자</div>
        <div class="participant-toggle-list" id="participant-toggle-list">
          ${ev.members.map(m => `<button type="button" class="participant-toggle-chip${participantIds.includes(m.id) ? ' active' : ''}" data-id="${m.id}">${m.emoji} ${escapeHtml(m.name)}</button>`).join('')}
        </div>
        <div class="participant-toggle-actions">
          <button type="button" id="btn-select-all" class="btn btn-ghost">전체 선택</button>
          <button type="button" id="btn-deselect-all" class="btn btn-ghost">전체 해제</button>
        </div>
        <details id="advanced-options">
          <summary>금액 직접 조정</summary>
          <div id="fixed-amount-rows">
            ${ev.members.map(m => `
              <div class="fixed-amount-row">
                <span>${m.emoji} ${escapeHtml(m.name)}</span>
                <input class="input input-fixed-amount" type="text" inputmode="numeric" data-id="${m.id}" placeholder="자동" value="${fixedAmounts[m.id] !== undefined ? fixedAmounts[m.id].toLocaleString(CONFIG.CURRENCY_LOCALE) : ''}" />
              </div>
            `).join('')}
          </div>
          <div id="fixed-preview" class="field-hint"></div>
        </details>
        <input id="input-item-memo" class="input" type="text" placeholder="메모 (선택)" value="${editing ? escapeAttr(editing.memo || '') : ''}" />
        <div id="item-form-error" class="field-error"></div>
        <div class="item-form-actions">
          <button type="button" id="btn-save-item" class="btn btn-primary">${editing ? '수정 완료' : '항목 추가'}</button>
          ${editing ? '<button type="button" id="btn-cancel-edit" class="btn btn-ghost">취소</button>' : ''}
        </div>
      </div>
    `;
  }

  function readFixedAmounts() {
    const result = {};
    document.querySelectorAll('.input-fixed-amount').forEach(input => {
      const value = parseAmountInput(input);
      if (input.value.trim() !== '' && value > 0) {
        result[input.dataset.id] = value;
      }
    });
    return result;
  }

  function updateFixedPreview(ev) {
    const previewEl = document.getElementById('fixed-preview');
    if (!previewEl) return;
    const amount = parseAmountInput(document.getElementById('input-item-amount'));
    const fixedAmounts = readFixedAmounts();
    const fixedSum = Object.values(fixedAmounts).reduce((s, v) => s + v, 0);
    const participantIds = Array.from(selectedItemParticipants);
    const remainderParticipants = participantIds.filter(id => fixedAmounts[id] === undefined);

    if (fixedSum > amount) {
      previewEl.textContent = '고정 금액 합이 총액을 초과합니다';
      previewEl.classList.add('field-error');
      return;
    }
    previewEl.classList.remove('field-error');

    if (remainderParticipants.length === 0) {
      previewEl.textContent = '';
      return;
    }
    const remaining = amount - fixedSum;
    const perPerson = Math.floor(remaining / remainderParticipants.length);
    previewEl.textContent = `잔여 ₩${remaining.toLocaleString(CONFIG.CURRENCY_LOCALE)}를 ${remainderParticipants.length}명이 나눔 → 1인 ₩${perPerson.toLocaleString(CONFIG.CURRENCY_LOCALE)}`;
  }

  function renderItemCard(item, ev, index, total) {
    const participantEmojis = ev.members.map(m => {
      const active = item.participantIds.includes(m.id);
      return `<span class="${active ? '' : 'item-participant-inactive'}">${m.emoji}</span>`;
    }).join('');
    const hasFixed = item.fixedAmounts && Object.keys(item.fixedAmounts).length > 0;
    return `
      <div class="item-card" data-id="${item.id}">
        <div class="item-card-top">
          <span class="item-emoji">${escapeHtml(item.emoji || '🧾')}</span>
          <span class="item-name">${escapeHtml(item.name)}</span>
          <span class="item-amount">₩${item.amount.toLocaleString(CONFIG.CURRENCY_LOCALE)}</span>
        </div>
        <div class="item-participants">${participantEmojis}${hasFixed ? renderBadge('직접 조정', 'primary') : ''}</div>
        <div class="item-card-actions">
          <button type="button" class="item-order-btn" data-id="${item.id}" data-direction="up" ${index === 0 ? 'disabled' : ''}>▲</button>
          <button type="button" class="item-order-btn" data-id="${item.id}" data-direction="down" ${index === total - 1 ? 'disabled' : ''}>▼</button>
          <button type="button" class="item-edit-btn btn btn-ghost" data-id="${item.id}">수정</button>
          <button type="button" class="item-delete-btn" data-id="${item.id}">✕</button>
        </div>
      </div>
    `;
  }

  function formatAmountInput(inputEl) {
    const digits = inputEl.value.replace(/[^\d]/g, '');
    inputEl.value = digits ? Number(digits).toLocaleString(CONFIG.CURRENCY_LOCALE) : '';
  }

  function parseAmountInput(inputEl) {
    return Number(inputEl.value.replace(/[^\d]/g, '')) || 0;
  }

```

Note: `item.emoji` is interpolated with `escapeHtml()` here (unlike Stage 1's member emoji, which is safe unescaped because it only ever comes from the fixed `CONFIG.MEMBER_EMOJIS` list) — item emoji can come from the free-text `#input-item-emoji-custom` field, so it must always be escaped.

- [ ] **Step 3: Register the route**

Find `// ====== 라우트 등록 & 초기 실행 ======` and the lines directly below it:

```javascript
  registerRoute('/', renderHome);
  registerRoute('/event/:id', renderEventSettings);
  window.addEventListener('DOMContentLoaded', router);
```

Replace with:

```javascript
  registerRoute('/', renderHome);
  registerRoute('/event/:id', renderEventSettings);
  registerRoute('/event/:id/items', renderItemsEntry);
  window.addEventListener('DOMContentLoaded', router);
  if (CONFIG.DEV_MODE) {
    window.addEventListener('DOMContentLoaded', runRegressionTests);
  }
```

- [ ] **Step 4: Verify manually in the browser**

At `http://localhost:8123/index.html`: create an event, add 2+ members, click "결제 입력으로 →" — confirm you land on `#/event/{id}/items` and see the empty item list + add form. Add an item (name, amount, leave all participants selected) — confirm it appears in the list with the correct formatted amount and all-participant emoji row. Add a second item, use the ▲/▼ buttons to reorder, confirm the list order updates and the boundary buttons (`▲` on the first item, `▼` on the last) are disabled. Delete an item via the ✕ button — confirm the `confirmDialog` shows and deletion only happens on confirm. Reload the page — confirm items persisted. Check the bottom bar shows correct per-member running totals.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Stage 2: 결제 입력 화면 (목록·추가/수정 폼·삭제·순서 변경·하단 요약바)"
```

---

### Task 5: 정산 결과 화면 — 표 · 송금 요약 · 계좌 복사 · 정산 완료

**Files:**
- Modify: `index.html` (`<style>` block: new CSS section; `<script>` block: new "화면: 정산 결과" section; route registration)

**Interfaces:**
- Consumes: `store`, `escapeHtml`, `showToast`, `calcSettlement` (Tasks 1–2), `registerRoute` (Stage 1)
- Produces: `renderSettlementResult(params)` (registers as `/event/:id/result`)

- [ ] **Step 1: Add CSS for the result screen**

Find the `/* ====== 반응형 보강 ====== */` comment (same anchor as Task 4 Step 1 — this CSS goes after Task 4's item-entry CSS, still before the responsive block). Insert:

```css
  /* ====== 정산 결과 ====== */
  .settlement-table-wrap {
    overflow-x: auto;
    border-radius: 16px;
    background: #fff;
    box-shadow: 0 2px 8px rgba(0,0,0,0.06);
  }
  .settlement-table {
    border-collapse: collapse;
    width: 100%;
    font-size: 13px;
    white-space: nowrap;
  }
  .settlement-table th, .settlement-table td {
    padding: 8px 10px;
    text-align: right;
    border-bottom: 1px solid #F0E8DC;
  }
  .settlement-table th:first-child,
  .settlement-table td:first-child {
    text-align: left;
    position: sticky;
    left: 0;
    background: #fff;
    z-index: 1;
  }
  .settlement-table tr.settlement-total-row td,
  .settlement-table tr.settlement-total-row th {
    background: #3D6FE0;
    color: #fff;
    font-weight: 700;
  }
  .settlement-table tr.settlement-total-row td:first-child {
    background: #3D6FE0;
  }
  .settlement-cell-empty { color: #C9BCAC; }

  .transfer-card { display: flex; flex-direction: column; gap: 4px; }
  .transfer-row { display: flex; align-items: center; justify-content: space-between; padding: 6px 0; }
  .transfer-owner-row { color: #4CAF50; font-weight: 700; }
  .transfer-amount { font-weight: 700; }

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

```

- [ ] **Step 2: Add `renderSettlementResult`**

Find `// ====== 라우트 등록 & 초기 실행 ======` (same anchor as Task 4 Step 3, now preceded by Task 4's items-screen code). Insert the entire new section immediately **before** it:

```javascript
  // ====== 화면: 정산 결과 ======
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
                ${ev.members.map(m => `<th>${m.emoji}</th>`).join('')}
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
      btn.addEventListener('click', () => { showToast('Stage 3에서 열립니다'); });
    });
  }

```

- [ ] **Step 3: Register the route**

Find (this is the version left by Task 4 Step 3):

```javascript
  registerRoute('/', renderHome);
  registerRoute('/event/:id', renderEventSettings);
  registerRoute('/event/:id/items', renderItemsEntry);
  window.addEventListener('DOMContentLoaded', router);
```

Replace with:

```javascript
  registerRoute('/', renderHome);
  registerRoute('/event/:id', renderEventSettings);
  registerRoute('/event/:id/items', renderItemsEntry);
  registerRoute('/event/:id/result', renderSettlementResult);
  window.addEventListener('DOMContentLoaded', router);
```

- [ ] **Step 4: Verify manually in the browser**

At `http://localhost:8123/index.html`: navigate to an event with a few items, click "정산 결과 보기 →" — confirm the table renders with correct per-member/per-item values matching the items screen's running totals, the total row is highlighted, and the grand total is correct. Exclude a member from one item (edit it on the items screen) then return to the result screen — confirm that cell shows `—`. Click "복사" (if account info is filled in) — confirm a toast appears. Click "정산 완료" — confirm the button becomes "정산 완료됨 ✓" and is disabled, and the home screen's badge changes to "정산 완료". Go back to the items screen and edit any item — confirm the home badge reverts to "작성 중" and the result screen's button becomes clickable again.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Stage 2: 정산 결과 화면 (표·송금 요약·계좌 복사·정산 완료)"
```

---

### Task 6: 반응형 마감 + 완료 기준 전체 검증 + CLAUDE.md 문서 업데이트

**Files:**
- Modify: `index.html` (small responsive CSS additions)
- Modify: `CLAUDE.md` ("Project status" section; "Settlement calculation engine — core invariants" section)

**Interfaces:**
- Consumes: entire Stage 2 implementation (Tasks 1–5)
- Produces: none (closing task)

- [ ] **Step 1: Add responsive tweaks**

Find the existing `@media (max-width: 400px)` block (added in Stage 1, currently containing `.card-section` and `.emoji-picker` rules). Add two more rules inside it:

```css
  @media (max-width: 400px) {
    .card-section { padding: 12px; }
    .emoji-picker { grid-template-columns: repeat(5, 1fr); }
    .item-summary-members { font-size: 12px; }
    .settlement-table th, .settlement-table td { padding: 6px 8px; }
  }
```

- [ ] **Step 2: Full regression pass — completion criteria from the spec**

Run `python -m http.server 8123` from the project root (if not already running) and navigate to `http://localhost:8123/index.html`. Run through spec §10 in order, starting from `localStorage.clear()`:

1. **회귀 테스트**: run `JSON.stringify(runRegressionTests())` in the console — confirm every entry `"pass":true` (seed A ₩39,820/₩199,100, seed B ₩54,800/₩274,000, all unit tests including the Task 2 ceil10/ceil100/fallback additions).
2. **시드 A 수동 입력**: create an event, add the 5 members (인우/찬우/정표/정우/은찬, owner=정우), add all 10 seed-A items via the UI with the exact amounts from the spec, go to the result screen — confirm the table matches: every member's total row shows ₩39,820 and the grand total shows ₩199,100.
3. **참여자 제외**: edit one item to deselect one member — confirm that member's cell shows `—` on the result screen and the total row recalculates correctly.
4. **금액 직접 조정**: on an item's "금액 직접 조정" section, enter a fixed amount for one participant that's less than the total — confirm the preview text updates to show the remaining amount split across the others; then enter a fixed amount exceeding the total — confirm the error text appears and the item cannot be saved with that value.
5. **정산 완료 → draft 복귀**: click "정산 완료" on the result screen — confirm the home card badge changes to "정산 완료". Go back to the items screen and edit any item — confirm the home card badge reverts to "작성 중" and the result screen's "정산 완료" button becomes clickable again.
6. **375px 레이아웃**: verify the settlement table has a working horizontal scrollbar and the first column (항목명) stays fixed in place while scrolling horizontally, with no page-level horizontal overflow. (Chrome automation cannot resize this environment's viewport reliably — if `resize_window` doesn't change the actual rendered viewport, use a same-origin `<iframe>` sized to 375px via `javascript_tool` and check `iframe.contentDocument.documentElement.scrollWidth <= 375`, the same workaround used in Stage 1's Task 6.)

If any criterion fails, investigate and fix the root cause before proceeding — don't note it and move on.

- [ ] **Step 3: Update CLAUDE.md's "Project status"**

Find:

```markdown
Stage 1 (skeleton: hash routing, CONFIG, event/member CRUD) is implemented in `index.html`. Stages 2-5 (settlement engine, receipts, Firebase sync, PWA polish) are not yet implemented — see `오내총_PRD_v1.md` §9 and `docs/superpowers/specs/` for design docs, and `docs/superpowers/plans/` for implementation plans.
```

Replace with:

```markdown
Stage 1 (skeleton: hash routing, CONFIG, event/member CRUD) and Stage 2 (item entry, settlement engine, result table) are implemented in `index.html`. Stages 3-5 (receipts, Firebase sync, PWA polish) are not yet implemented — see `오내총_PRD_v1.md` §9 and `docs/superpowers/specs/` for design docs, and `docs/superpowers/plans/` for implementation plans.
```

- [ ] **Step 4: Update CLAUDE.md's settlement-engine section with implementation specifics**

Find:

```markdown
- **Mandatory integrity check on every save**: for every item, `sum(per-participant amounts) === item.amount` exactly (zero won of drift). If it fails, block the save and identify the offending item — don't silently correct it.
```

Replace with:

```markdown
- **Mandatory integrity check**: `calcSettlement(event)` asserts `sum(per-participant amounts) === item.amount` for every item and collects violations into `errors[]` (codes: `NO_PARTICIPANT`, `FIXED_EXCEEDS`, `FIXED_NOT_PARTICIPANT`, `INTEGRITY_MISMATCH`). Saving an item is always allowed even with errors present — only transitioning `status` to `'settled'` is blocked while `errors.length > 0` (shown as a warning banner + disabled 정산 완료 button on the result screen).
```

Then find:

```markdown
- **Regression test requirement (Stage 2 completion criterion)**: reproduce the real spreadsheet seed data from the PRD and assert totals match exactly (₩199,100 / ₩274,000). Hardcode this seed data as a test fixture — don't skip this check.
```

Add immediately after it:

```markdown
- **Implementation reference (Stage 2)**: the engine is the pure function `calcSettlement(event)` in `index.html`, calling `distributeRemainder`/`distributeFloor1` per item. `CONFIG.ROUNDING` selects `'floor1'` (default) | `'ceil10'` | `'ceil100'`. The floor-remainder ('끝전') is credited to the 총무 if they participate in that item, otherwise to the first participating member in `event.members` array order. `ceil10`/`ceil100` round non-owner shares up to the nearest 10/100 won and derive the 총무's share by subtraction; if that subtraction would go negative, the item falls back to `floor1` and logs a `console.warn` identifying the item.
```

- [ ] **Step 5: Commit**

```bash
git add index.html CLAUDE.md
git commit -m "Stage 2: 반응형 마감, 완료 기준 전체 검증, CLAUDE.md 정산 규칙 기록"
```
