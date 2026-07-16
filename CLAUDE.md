# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

Stage 1 (skeleton: hash routing, CONFIG, event/member CRUD) is implemented in `index.html`. Stages 2-5 (settlement engine, receipts, Firebase sync, PWA polish) are not yet implemented — see `오내총_PRD_v1.md` §9 and `docs/superpowers/specs/` for design docs, and `docs/superpowers/plans/` for implementation plans.

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

## Data storage model (hybrid — important, don't collapse into one store)

| Data | Location | Why |
|---|---|---|
| Events, members, items, settlement inputs, account info | **Firebase Firestore** | needs cross-device sync + shareable read links |
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
- **Mandatory integrity check on every save**: for every item, `sum(per-participant amounts) === item.amount` exactly (zero won of drift). If it fails, block the save and identify the offending item — don't silently correct it.
- If fixed amounts for an item sum to more than the item's total, block save immediately at input time (don't wait for the aggregate check).
- Edge cases with defined behavior (see PRD §8 table) — e.g., an item with zero participants is blocked at save, an item where only the 총무 participates is allowed (generates no charges), zero-won items are allowed (for record-keeping).
- **Regression test requirement (Stage 2 completion criterion)**: reproduce the real spreadsheet seed data from the PRD and assert totals match exactly (₩199,100 / ₩274,000). Hardcode this seed data as a test fixture — don't skip this check.

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
