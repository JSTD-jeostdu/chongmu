# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

Stage 1 (skeleton: hash routing, CONFIG, event/member CRUD), Stage 2 (item entry, settlement engine, result table), Stage 3 (감성 영수증: photo attach, dot-font receipt DOM, html2canvas capture/share), and Stage 4+5 (Firebase Auth + Firestore sync, public share view, PWA manifest) are implemented in `index.html`. See `오내총_PRD_v1.md` §9 and `docs/superpowers/specs/` for design docs, and `docs/superpowers/plans/` for implementation plans.

## What this app is

**오늘은 내가 총무 (오.내.총)** — a webapp for a group's designated payer ("총무") to log shared expenses, compute per-person settlement amounts, and generate stylized "감성 영수증" (thermal-receipt-style) images per person to share in KakaoTalk, including one highlight photo per receipt.

## Hard technical constraints (do not deviate without asking)

- **Single `index.html` file.** Vanilla HTML/CSS/JS only — no build tools, no npm, no bundler, no framework.
- **CDN-only dependencies**: Firebase SDK (compat mode), `html2canvas`, web fonts (e.g. Galmuri/DungGeunMo for the receipt's dot-matrix look).
- **Hash-based routing** (`location.hash`) since there's only one HTML file. Routes: `/` (event list), event create/settings, item entry, settlement result, receipt view, and `/#/view/{shareId}` (public read-only view, no login).
- **All tunable constants live in one `CONFIG` object** — rounding policy, receipt caption pool, color palette, image compression params, Firebase config. Don't scatter magic numbers/strings elsewhere.
- **Comments are written in Korean** (matches PRD and target audience).
- **Deployment target**: GitHub Pages.
- **Mobile-first, responsive**; PWA manifest + icons are in scope for v1, but a service worker is explicitly **not** (Firestore offline persistence substitutes for it).

## Photo storage & receipt capture (Stage 3 implementation reference)

- `photoStore` (in `index.html`, parallel to `store`) is the only code path that touches `localStorage[CONFIG.PHOTOS_KEY]`. API: `get(eventId, memberId)`, `set(eventId, memberId, dataUrl)` (throws `QuotaExceededError` on failure — callers must catch it), `remove(eventId, memberId)`, `removeByEvent(eventId)`, `listByEvent(eventId)`, `usage()` (KB estimate). `event.*` objects never reference photos directly — the two are joined only by the `"{eventId}:{memberId}"` key convention.
- Photos are resized (`resizeImageToCanvas`, long edge capped at `CONFIG.PHOTO_MAX_EDGE`) and filtered (`applyPhotoFilter`: `'original'|'grayscale'|'dither'`) via `<canvas>` pixel manipulation at save time — never via CSS `filter`, which `html2canvas` does not reliably capture. Only the final filtered/compressed result is persisted; the resized-but-unfiltered intermediate lives only in the module-scope `pendingPhotoCanvas` variable during the attach modal's session and is never written to `localStorage`.
- Any element that must not appear in a captured receipt PNG (e.g. the account-copy button) must carry a `data-no-capture` attribute — `captureReceiptToBlob`'s `html2canvas` call excludes those via `ignoreElements`.
- Every capture (`captureReceiptToBlob`) must `await document.fonts.ready` first, or the Galmuri dot font may not have finished loading/rasterizing at capture time.
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

| Data | Location | Why |
|---|---|---|
| Events, members, items, settlement inputs, account info | **Firebase Firestore when signed in, `localStorage` otherwise** (always mirrored to `localStorage` too — see "Firebase sync & share view" above) | needs cross-device sync + shareable read links, but must work with zero setup/no account |
| Highlight photos (base64 JPEG) | **`localStorage`** only, keyed `onct_photos: "{eventId}:{memberId}"` | avoids Firebase Storage cost/complexity |
| Rounding/UI prefs | `localStorage`, key `onct_prefs` | device-local, no sync needed |

Consequences that matter for implementation:
- Receipt images (which include the photo) can only be generated on the **총무's own device**, since photos never leave `localStorage`. The shared read-only view never shows photos — only text settlement data.
- If the 총무 switches devices, photos are lost but Firestore data is not. Surface a one-time notice about this.
- Firestore security rules: **only the event's creator (`ownerUid`) can write**; anyone with the `shareId` can read via the `/view/{shareId}` route (no auth required for reads).
- Settlement results are **never persisted** — always recomputed from `items` on the fly, so `items` is the single source of truth and results can't drift out of sync with edits.
- Writes should be debounced (~800ms) with a visible save-state indicator ("저장됨 ✓"). Single-editor model means last-write-wins is an acceptable sync strategy (no merge logic needed).

## Settlement calculation engine — core invariants

This is the trickiest part of the app; treat these as correctness requirements, not suggestions:

- Per item: participants with a **fixed assigned amount** pay exactly that; the remainder (`amount − sum(fixed amounts)`) splits evenly across the remaining participants.
- Default rounding: **floor to the won**, and the leftover from flooring is absorbed into the 총무's own share (configurable in `CONFIG` to instead round participant shares up to nearest 10/100 won, with the excess deducted from the 총무's share instead).
- **Mandatory integrity check**: `calcSettlement(event)` asserts `sum(per-participant amounts) === item.amount` for every item and collects violations into `errors[]` (codes: `NO_PARTICIPANT`, `FIXED_EXCEEDS`, `FIXED_NOT_PARTICIPANT`, `INTEGRITY_MISMATCH`). Saving an item is always allowed even with errors present — only transitioning `status` to `'settled'` is blocked while `errors.length > 0` (shown as a warning banner + disabled 정산 완료 button on the result screen).
- If fixed amounts for an item sum to more than the item's total, block save immediately at input time (don't wait for the aggregate check).
- Edge cases with defined behavior (see PRD §8 table) — e.g., an item with zero participants is blocked at save, an item where only the 총무 participates is allowed (generates no charges), zero-won items are allowed (for record-keeping).
- **Regression test requirement (Stage 2 completion criterion)**: reproduce the real spreadsheet seed data from the PRD and assert totals match exactly (₩199,100 / ₩274,000). Hardcode this seed data as a test fixture — don't skip this check.
- **Implementation reference (Stage 2)**: the engine is the pure function `calcSettlement(event)` in `index.html`, calling `distributeRemainder`/`distributeFloor1` per item. `CONFIG.ROUNDING` selects `'floor1'` (default) | `'ceil10'` | `'ceil100'`. The floor-remainder ('끝전') is credited to the 총무 if they participate in that item, otherwise to the first participating member in `event.members` array order. `ceil10`/`ceil100` round non-owner shares up to the nearest 10/100 won and derive the 총무's share by subtraction; if that subtraction would go negative, the item falls back to `floor1` and logs a `console.warn` identifying the item.

## Implementation stages (PRD §9)

Work is meant to proceed in this order; each stage has its own completion gate. Don't jump ahead to Firebase sync (Stage 4) or receipt polish (Stage 3/5) before the settlement engine (Stage 2) passes its regression test.

1. **Stage 1** — Skeleton: hash routing, `CONFIG`, event CRUD + member management, localStorage only (no Firebase yet).
2. **Stage 2** — Item entry + settlement engine + integrity validation + result table. Gate: seed-data regression test matches exactly.
3. **Stage 3** — Receipt generation: design, photo attach/compress, `html2canvas` export, Web Share API. Gate: verify rendering quality when sent through KakaoTalk.
4. **Stage 4** — Firebase: Auth (Google), Firestore sync, Security Rules, public share view. Gate: cross-device restore + unauthenticated read access both work.
5. **Stage 5** — Polish: PWA manifest, responsive cleanup, remaining edge cases, in-app notices. Gate: all success criteria in PRD §2.2 pass.

## Data model reference

See PRD §5 for the full Firestore schema (`events` collection with nested `members[]` and `items[]`, plus the `localStorage` shape for photos/prefs). Key shape notes:
- `items[].fixedAmounts` is only present when using the "manual amount" distribution mode for that item; otherwise distribution is even split among `participantIds`.
- Amounts are always integer won — never store or compute in floating point won-fractions.
