---
target: web/public (index.html, scripts.js, styles.css)
total_score: 23
p0_count: 1
p1_count: 2
timestamp: 2026-06-22T22-26-12Z
slug: web-public-index-html
---
## Design Health Score

| # | Heuristic | Score | Key Issue |
|---|-----------|-------|-----------|
| 1 | Visibility of System Status | 2/4 | Save/delete confirmations are `alert()` popups (scripts.js:6007, 6203) — blocking, easily missed, not inline |
| 2 | Match System / Real World | 3/4 | Correct VTM domain vocabulary (Хроника, Маскарад, Торпор) for GM audience; nav relies on emoji as functional icons |
| 3 | User Control and Freedom | 2/4 | No undo after delete anywhere; some deletes double-confirm (module, scripts.js:3264-3265) but no recovery path exists |
| 4 | Consistency and Standards | 3/4 | Strong token discipline overall, but 4 separate button-class families (`.btn-submit`, `.chr-modal-btn`, `.modp-*-btn`, `.cdet-*-btn`) duplicate the same job |
| 5 | Error Prevention | 2/4 | Validation is mostly post-submit via `alert()` (index.html:1491-1495), not inline/live |
| 6 | Recognition Rather Than Recall | 3/4 | Filters visible, not hidden; `field-tip` tooltips explain jargon inline — but only on the new-city form |
| 7 | Flexibility and Efficiency | 2/4 | No keyboard shortcuts beyond Enter/Escape; no bulk actions despite repetitive GM workflows (thread-closing, rumor-marking) |
| 8 | Aesthetic and Minimalist Design | 3/4 | Generally restrained; Tools page crams 7 tabs into one row, borderline dense |
| 9 | Error Recovery | 2/4 | Raw JS exceptions surfaced via `alert('Ошибка: ' + e.message)` (scripts.js:5343, 6037, 6120, 6227) — not actionable |
| 10 | Help and Documentation | 1/4 | No in-app help surface; `field-tip` tooltips exist only on one form |
| **Total** | | **23/40** | **Acceptable — significant improvements needed, esp. error handling/help/flexibility** |

## Anti-Patterns Verdict

**LLM assessment**: Does not read as AI slop. DESIGN.md's named rules (Whisper-Until-Touched, Full-Border Rule, No-Material-Shadow) are actually load-bearing in the CSS — `color-mix()`-based clan-tint surfaces, a monotonic `--fs-*` type scale, `:focus-visible` rules, `prefers-reduced-motion` handling, and `pointer:coarse` touch-target gating all show real craft, not vibes. The one place a side-stripe pattern appears outside the documented `.nav-item` exception (`.char-card::before`) is a correct hover-only full-bleed accent, not the banned decorative strip. What breaks the illusion: the app falls back to native `alert()`/`confirm()` for dozens of save/error/delete flows — a flat, undesigned moment colliding with an otherwise deliberate dark-archive aesthetic, especially jarring at the highest-stakes action (deleting irreplaceable character history).

**Deterministic scan** (`detect.mjs` on `index.html`, `scripts.js`, exit code 2, 4 findings):
- 2× `broken-image` warnings (`#modal-img-thumb` index.html:880, `#lightbox-img` index.html:930) — **false positive**: both are intentionally-empty `<img>` placeholders populated dynamically by JS when a modal/lightbox opens, not shipped-broken images.
- 1× `single-font` warning (index.html:8) — **false positive**: the detector only inspected the Google Fonts `<link>` tag, which happens to load Cinzel; it missed that `styles.css` defines four families (Cinzel Decorative/Cinzel/Cormorant Garamond/Share Tech Mono) plus a fifth (`Inkulinati`/`Inknut Antiqua` for `.page-sub`, loaded locally, undocumented in DESIGN.md — a real but different issue, see Minor Observations).
- 1× `em-dash-overuse` (6 em-dashes in body copy) — **likely false positive for this project**: em-dash is a standard Russian typographic convention, not an AI cadence tell, in Cyrillic prose.

No reliable user-visible browser overlay was available this run: no browser automation tool is present in this environment, so Assessment B is CLI-only (no live-server injection, no screenshot evidence).

## Overall Impression

A genuinely designed, on-brand dark-archive UI undermined by inconsistent treatment of its highest-stakes moments. The visual system (tokens, motion, color) is real and disciplined; the interaction layer (errors, deletes, help) still leans on 1990s browser primitives. Biggest opportunity: route every destructive action and every error/success message through the app's own styled modal/toast vocabulary instead of `alert()`/`confirm()` — this single change would lift heuristics #1, #3, #5, and #9 simultaneously.

## What's Working

1. **Token discipline is load-bearing, not aspirational.** `color-mix()`-driven clan-tinted surfaces (styles.css:2008-2046, `var(--clan-tint, var(--accent))`) and the monotonic `--fs-*` scale show the documented design system is actually implemented in the CSS, not just written down in DESIGN.md and ignored.
2. **Accessibility groundwork beyond cosmetics.** `:focus-visible` outlines (styles.css:1734-1748), `prefers-reduced-motion` handling (1750-1760), and 44px touch targets gated behind `@media (pointer: coarse)` (1715-1732) show real testing against non-mouse input — rare in AI-assisted builds.
3. **The one side-stripe exception is correctly scoped.** `.char-card::before`'s hover-revealed full-bleed accent and `.nav-item`'s documented active-indicator are the only two stripe-shaped patterns in the codebase, and both are functional/deliberate rather than decorative — the Full-Border Rule is actually being honored, not just stated.

## Priority Issues

**[P0] Destructive actions inconsistently routed through native browser dialogs.**
- **Why it matters**: Character/thread/NPC deletes fall back to `confirm()`/`alert()` (scripts.js:1873-1885) while module/chronicle deletes use the app's own styled danger modal (index.html:374-395). For a tool whose entire purpose is protecting years of irreplaceable character history, a GM can blow through a generic OS dialog without registering what's being destroyed.
- **Fix**: Route all deletes through the existing `chr-modal` danger-confirm pattern; remove every `confirm()`/`alert()` call tied to a destructive action.
- **Suggested command**: `/impeccable harden`

**[P1] Modals are invisible to assistive tech.**
- **Why it matters**: No `role="dialog"` or `aria-modal="true"` exists anywhere across ~10 modal overlays in index.html/scripts.js, and no focus-trap logic was found. A screen-reader user gets no signal a dialog opened.
- **Fix**: Add `role="dialog" aria-modal="true" aria-labelledby` to every `.modal-box`/`.chr-modal`, plus focus-trap on open/close.
- **Suggested command**: `/impeccable harden`

**[P1] Errors leak raw JS exceptions instead of diagnosing the problem.**
- **Why it matters**: `alert('Ошибка: ' + e.message)` patterns (scripts.js:5343, 6037, 6120, 6227) dump technical strings instead of plain-language, field-located guidance.
- **Fix**: Map known failure modes to Russian-language messages naming the problem and the next step; surface inline near the failing field, not in a blocking alert.
- **Suggested command**: `/impeccable clarify`

**[P2] Zero progressive disclosure on the app's most complex single screen.**
- **Why it matters**: The Log Session form (index.html:721-818) expands Модуль/Событие/Сцены/Последствия/Участники/Нити/Финал simultaneously — 7 full sections visible at once for the highest-frequency GM task.
- **Fix**: Collapse to an accordion or stepper; show Модуль + Событие by default, reveal the rest as filled.
- **Suggested command**: `/impeccable layout`

**[P3] Tools page tab bar overcrowded.**
- **Why it matters**: 7 tabs in one row (index.html:503-511), one labeled "🛠 Ещё" containing 5 unrelated subsections (new location, cross-city migration, close chronicle, rebuild index, sync registry) — a junk-drawer tab.
- **Fix**: Group into 2 tiers or a dropdown for the long tail.
- **Suggested command**: `/impeccable distill`

## Persona Red Flags

**Alex (Power User / the actual GM)**: No keyboard shortcuts for navigating the 11 sidebar sections (every switch is a mouse click on `.nav-item`); no bulk actions anywhere — can't multi-select threads to close or batch-mark rumors as told beyond one-by-one checkboxes (scripts.js:3774). For someone running this tool every session for years, this is the single biggest efficiency gap.

**Sam (Accessibility / screen-reader + keyboard)**: Beyond the missing `role="dialog"` (see P1 above), `.page-title` is correctly hidden via clip-rect (styles.css:291-303) but the *visible* `.page-sub` heading isn't connected to it via `aria-labelledby` — a screen reader gets a real H1 disconnected from what sighted users actually read as the title. Close buttons correctly carry `aria-label="Закрыть"`, so that part works.

## Minor Observations

- `.page-sub` (styles.css:306) declares `'Inknut Antiqua', serif` as fallback for a locally-loaded `Inkulinati` font, neither of which is in the Google Fonts `<link>` (index.html:8) nor documented in DESIGN.md's typography section — a real drift between system-as-documented (3 families) and system-as-shipped (5), with a silent fallback to default serif if the local font file fails to load.
- 4 separate button-class families doing the same job (`.btn-submit`, `.chr-modal-btn`, `.modp-*-btn`, `.cdet-*-btn`) — candidates for consolidation into one primitive next time `/impeccable distill` or `/impeccable layout` touches buttons.

## Questions to Consider

- If destructive deletes are the moment a GM is most likely to be tired or rushing through a session — why does the UI hand that exact moment to the one component (`confirm()`) that breaks the carefully built archive metaphor?
- The Log Session form is the highest-frequency, highest-cognitive-load task in the app — is the gothic-archive density serving the GM here, or fighting them at the one moment efficiency should win over atmosphere?
