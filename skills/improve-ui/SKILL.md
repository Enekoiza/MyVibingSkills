---
name: improve-ui
description: "Audit and improve a UI (component, page or app) against concrete UX pattern rules: hierarchy, states, motion, forms, overlays, accessibility, copy. Use when asked to improve, polish or review UI."
argument-hint: [component, page or path to review]
model: claude-sonnet-5-5
effort: medium
---

# /improve-ui

Turn a UI that "works" into one that feels deliberate and premium, using concrete, numeric rules instead of taste. Based on the designmotionhq UX pattern library (76 patterns), condensed into checkable rules. Works in any repo: the numbers below are defaults, and the project's own design system is never overridden without asking.

## Workflow

1. **Scope.** Identify what is being improved (a component, a screen, a flow, a whole app) and the stack (React, plain CSS, Tailwind, SwiftUI...). Read the relevant files before judging anything.
2. **Detect the existing design system.** Look for: token files (CSS custom properties in `:root`, `tokens.json`, `theme.ts`, SCSS variables), `tailwind.config.*` / `@theme` blocks, component-library themes (MUI, Chakra, Mantine, shadcn, Bootstrap variables), SwiftUI/Android theme files, or a clearly repeated set of values across components. Note its values for spacing, radius, colors, shadows, type scale and motion.
3. **Audit.** Walk the checklist sections below that apply to what is on screen. For every violation, note: where (file/line or element), which rule, the fix. While auditing, sort each finding into one of two buckets:
   - **Normal finding:** the rule does not contradict anything the project has deliberately defined (missing states, accessibility, a value that is off the project's own scale, hover-only actions, copy...). If the project has a scale, the fix uses the project's scale, not this skill's numbers.
   - **Design-system conflict:** the project deliberately defines something that differs from this skill's default (e.g. project radius scale 6/10/14 vs skill 4/8/12/16/24; dark background #000 in tokens vs skill #121212; transitions 400ms in theme vs skill 200–300ms; spacing on a 5px base vs 4px). Never silently change these. They go in the conflicts section of the report.
   - Accessibility minimums (contrast 4.5:1 / 3:1, focus visibility, 44px touch targets, not relying on color alone) are always normal findings, even if the design system violates them, because they are not a matter of taste. Mark them clearly as coming from the design system so the user knows the fix touches tokens.
4. **Prioritise** normal findings:
   - **P1 – Broken/risky:** accessibility failures, data loss risk, missing states (error/empty/loading), destructive actions unguarded, lying UI ("Saved" when it wasn't, rounded ratings, truncated chart axes).
   - **P2 – Feels cheap:** inconsistent spacing/radius/shadows, no hierarchy, wrong motion timing, hover-only actions, generic copy.
   - **P3 – Polish:** staggers, micro-interactions, delight moments.
5. **Report in the conversation and stop.** Write the report directly in your reply using the output format at the end of this skill. Never write it to a file. Number every finding so the user can pick them. For each design-system conflict, show both options side by side: **A) keep existing** (what stays, why it is still coherent) and **B) apply skill rule** (the new value, what it would look like, and the blast radius: which token changes and how many components/screens it affects). End by asking which fixes to apply and, for each conflict, A or B. Do NOT modify any code in this turn.
6. **Apply after the user answers.** Implement only what the user chose, P1 first. For conflicts resolved as B, change the value at the token/theme level (not per component) so it cascades consistently; for A, leave it and do not raise the same conflict again in this session. Then give a short summary of what changed (files touched, one line per fix).
7. **Verify.** Re-read the changed code; if a preview/browser is available, look at it (light and dark, desktop and mobile width, keyboard Tab through it).

Exception: if the user explicitly says to apply fixes directly ("just fix it", "don't ask", "apply them directly"), skip the wait for normal findings and go straight to applying them. Design-system conflicts still default to A (keep existing) unless the user said otherwise, and are listed at the end of the summary as "not changed, would conflict with your design system".

If the project has no design system at all, there are no conflicts: propose a minimal token set as a finding and build the other fixes on it.

Rules of thumb: fix the system before the instance (tokens > one-off values); prefer the browser/CSS doing the work over mirrored JS state; never trade accessibility for looks. For non-web stacks (SwiftUI, Android, Flutter) apply the principles and numbers, translating the CSS-specific implementation notes to the platform's equivalents.

---

## 1. Visual system (decide once, reuse everywhere)

**Tokens**
- Name tokens by meaning, not value: `color-primary`, `space-md`, never `blue-500` or `spacing-16` in component code.
- Three layers: primitives (raw scale) → semantic (brand, error, surface) → component. One primitive change should cascade.
- No raw hex or pixel values inline in components.
- Color scales numbered 100–900; spacing on a 4px scale (4, 8, 12, 16, 24, 32, 48, 64). Snap strays (7px, 13px, 23px) to the scale.
- Optional proportional scale: base × 1.618 (8 → 13 → 21 → 34 → 55; type 16 / 26 / 42 / 68), rounded to the grid. Layout split ≈ 62/38.

**Hierarchy**
- Primary element ≈ 2× body text size. Weights: headings ~700–800, body 400, captions 300.
- One accent color, reserved for the single most important action (and selection state). Everything else neutral.
- Secondary text via lower opacity (~55–70%), not a new color.
- Most important element gets the most whitespace. Don't give everything equal emphasis.
- Von Restorff: emphasise exactly one thing per view (target plan, primary CTA) with scale + elevation + color, against a calm uniform baseline. Two highlighted things cancel out.

**Grouping & layout**
- Proximity: gap inside a group < gap between groups (e.g. ~12px between related fields, ~40px between sections). Use space before borders/dividers.
- Gestalt: similarity (shared color/shape groups items), continuity (align on a shared axis), common region (card/border groups distant items), figure-ground (dim/blur behind modals).
- 12-column grid; ratios 4:8, 6:6, 3:9. Gutters: 8px dense, 24px balanced, 40px editorial. Collapse 12 → 6 → 4 → 1 across breakpoints. Break the grid only on purpose, once it is established.
- Alignment: labels/IDs left, numbers/dates right, icons centered on the text line. Avoid centering data.
- Serial position: most important items first and last; the middle is forgotten (logo first, primary CTA last in nav).

**Radius**
- One scale: 4 · 8 · 12 · 16 · 24. Scale with element size: tooltip 4, input 8, card 12, modal 16, panel 24.
- Nested corners: inner radius = outer radius − padding.
- Small radius reads corporate, large reads friendly: match the brand.

**Shadows & depth**
- Layer shadows: tight contact (~0 1px 3px) + mid (~12px blur) + wide ambient (~32px). Never one flat drop shadow for everything.
- Elevation encodes importance; strongest elevation for what matters most.
- Alternative (dense tools, Linear-style): no shadows; three stacked background values (base / surface / raised) + 1px hairline border (~8% white on dark, ~12% for cards). Hover changes surface value.
- Optional brand-tinted glow at low opacity.

**Dark mode**
- Base ≈ #121212, never #000. Each elevation step a lighter grey.
- Desaturate accents ~20%. Text soft off-white, never #FFF. Emphasis via white opacity tiers.
- Dark mode = swapping a token set, not inverting colors.

**Color & contrast**
- WCAG: 4.5:1 body text, 3:1 large text and focus rings/UI. Check "muted" greys in nav links and labels (they often sit at 1.5–2:1).
- Never color alone: pair with icon, label or pattern (~8% of men are color-blind).
- Red is a budget: destructive actions only (not logout, not decorative badges).
- Gradients: ≤60° of hue travel (neighbouring hues), lightness moving one direction, 2–3% noise to hide banding, use as ambient glow behind content rather than full-bleed surface; no body text on the mid-transition zone.

**Icons**
- One stroke weight (2px) across the set; one strategy (all outline or all filled).
- Snap to a 24px grid (16px dense). Same bounding box for every icon. Round shapes 5–8% larger than square ones to look equal.

**Density (tools/dashboards)**
- 13px text, ~32px rows, slightly tightened letter-spacing. Status as grey icons rather than colored pills. Show keyboard shortcuts next to actions.

---

## 2. Motion

- Entrances 200–300ms, ease-out. Exits faster (~150ms, ~40% shorter). Never symmetric.
- Tap/press feedback < 100ms. Hover feedback ~80ms; UI transitions in tools < 150ms, no bounce.
- Linear easing only for continuous motion (spinners, marquees). Springs/overshoot only where attention is wanted (presses, confirmations, notifications 500–800ms). Subtle, never wobbly.
- Stagger list/card entrances ~50ms (30ms blurs, 100ms crawls); revealed actions ~60ms.
- Use the same small set of curves everywhere. Tie easing to intent: ease-out enter, ease-in-out move, ease-in exit. Duration scale 100ms (micro) → 500ms (complex).
- Card hover: lift ~8px + growing shadow over ~200ms ease-out; image scale ~1.05 inside `overflow: hidden`. Never scale the whole card (shifts neighbours).
- Scroll effects: prefer CSS `animation-timeline: scroll()` / `view()` with `animation-range`, plus `position: sticky`, over JS scroll listeners.
- Respect `prefers-reduced-motion`.

---

## 3. Feedback & states

**Every data view has: loading, empty, error, success/populated, (partial/offline).** Missing one is a bug.

**Speed (Doherty threshold)**
- Visible response within 400ms of any action. < 200ms feels instant.

**Loading — pick by what you know**
- < 300ms: show nothing (a flash looks like a glitch).
- Known shape, > 300ms: skeleton that matches the real layout exactly, with shimmer. Don't mix skeletons and spinners in one view.
- Unknown duration, short (< ~3s): spinner, scoped to the thing loading (button, not whole page).
- Known progress, > 3s: progress bar with real meta (percent, time left).
- Reversible user actions (like, toggle, reorder, rename): optimistic UI — update instantly, reconcile, roll back cleanly on failure with a message.
- Never optimistic for payments, transfers, bookings or anything irreversible: hold the spinner until the server confirms, then show server truth (real IDs).

**Empty states** — four kinds: first run, no results, error, filtered-out; each with its own copy and CTA.
- Small illustration/icon so it reads intentional; warm human copy; one primary CTA that is the real next step ("+ New project", not "Try refreshing"). First run can show a ghost preview of a real item.

**Errors**
- Surface by severity: field → inline under the field; transient/connection → toast or banner; blocking → modal/prominent area.
- Every error has an exit: Retry, link to help, or expandable details. No dead-end "OK".
- Plain language: what broke, why, what to do. No raw "Error 500".

**Notifications (4 surfaces)**
- Toast (low, auto-dismiss, undo), banner (persistent until cleared), modal (blocks, must act), badge (passive count). Severity picks the surface; over-escalation trains users to ignore everything. Never stack modals.

**Toasts**
- Bottom-right desktop, top edge mobile, never center. Info ~4s, warning ~7s, critical stays until acknowledged. Max 3 visible, queue the rest. Close button, swipe to dismiss, pause timer on hover. Type = color + icon + accent border.

**Undo over confirm**
- Do the action immediately, offer undo (5–10s) with a visible draining countdown. Soft delete (trash ~30 days). Undo stack (Cmd+Z) where editing. Delayed send for messages.

**Psychology**
- Peak-end: design one delightful peak and a strong ending screen; the last step weighs most. Never end a flow on friction.
- Zeigarnik: visible unfinished progress (80% profile, one unchecked onboarding item) pulls users back — only for outcomes they actually want.

---

## 4. Forms & inputs

**Field states (six):** default, focus, error, success, disabled, loading — each designed.
- Label above (persistent), helper text below. Placeholder is never the label.
- Focus ring ≥ 3:1 contrast.
- Error = color + icon + message naming the fix.
- Success confirmed inside the field (green check), not a toast.
- Disabled: grayscale + not-allowed cursor. Loading: in-field spinner, blocks double submit. Don't fake disabled with opacity .5.

**Validation timing**
- Validate on blur, not per keystroke and not only on submit. After a field has errored, validate live so the error clears as soon as it is fixed.
- Client validation for speed, server validation for trust; recompute prices/totals server-side; multi-row writes in one transaction.
- CSS can drive it: `.field:has(:user-invalid) { border-color: var(--color-error) }` — use `:user-invalid`, not `:invalid`. Keep the submit button at full strength (see Submit buttons).

**Submit buttons**
- Don't disable submit with no explanation (drops out of tab order, tooltips can't fire, fails contrast). Keep it enabled, validate on click, highlight blocking fields, focus the first.
- Loading ≠ disabled: keep focus, show spinner, `aria-busy="true"`.

**Long forms**
- Chunk into steps grouped by meaning (Personal, Shipping, Payment, Review). Progress indicator. Validate within each step. Persist state across Back/refresh.

**Autosave**
- Debounce ~800ms after typing stops. Status as state machine: typing / saving / saved / offline / error — never show "Saved" unless it landed. Queue offline edits and replay in order. Merge or warn on concurrent edits. `beforeunload` guard for unsaved work.

**Settings**
- Low-risk toggles apply instantly; identity fields (email, password) need explicit Save/Cancel. Group by user task. Searchable. Modified indicator + per-setting reset. Advanced options collapsed. Danger zone at the bottom, delete requires typing the resource name.

**Specific inputs**
- *Masked input (cards, phones):* group digits (4-4-4-4), detect card brand from first digit, keep caret after auto-inserted separators, clean pasted separators, validate on blur, display formatted / store raw.
- *Password:* live requirements checklist + strength meter (entropy/length over symbol rules), show/hide eye toggle, allow paste, offer generated password.
- *OTP:* one string as source of truth, boxes are a view; paste distributes digits (strip non-digits); auto-advance, backspace goes back; `inputmode="numeric"` + `autocomplete="one-time-code"`; resend behind visible 30s countdown; wrong code shakes + clears, right code locks green.
- *Date picker:* presets first (Today, Yesterday, Last 7/30 days, Last quarter); custom range with hover preview, two months side by side; full keyboard (arrows, typing, Enter, Esc, PgUp/Shift+PgUp); full-screen sheet on mobile.
- *Range slider:* filled track = value; whole row is the hit area; snap to steps with ticks; value tooltip while dragging; two-thumb range with filled band; arrows/Home/End.
- *Toggle:* rail 2× knob; animate rail color + knob position + shadow + label together ~250ms ease-out; Space toggles, visible focus, `role="switch"`/`aria-checked`; async = optimistic flip, spinner in knob, roll back with shake + toast.
- *File upload:* dropzone reacts on drag-over (border, glow, copy); percent + time remaining; inline retry that keeps the file; show thumbnail, type, size; per-file progress/retry in queues.
- *Color picker:* OKLCH alongside hex; recents and saved palettes; live contrast badge; alpha previewed on a checkerboard over light and dark; generate tint/shade scale from one pick.
- *Star rating:* preview on hover ahead of cursor, revert on leave; fractional fills for averages (4.4 = 4 + 44%), never round up; stagger fill ~30ms per star; pair with average + distribution summary.

---

## 5. Overlays

**Choosing:** first ask "does it block the user?"
- Yes → **modal**: full scrim, one centered decision; critical/destructive only.
- No → **bottom sheet** (mobile default; drag handle, snap points half/full, drag-to-dismiss, scrim, body scroll locked, page stays visible), **drawer** (edge-anchored, navigation), **popover** (anchored to trigger, small ~200px, contextual).
- Scroll lock via CSS: `body:has(dialog[open]) { overflow: hidden }`.

**Dropdowns**
- Trigger: 48px target, visible caret, real hover state. Flip upward when no room. Arrows / Enter / Esc. Search field once > ~10 items. Open in ~150ms.

**Context menus**
- Measure and flip to stay in viewport, anchored to cursor. Group by intent with dividers; destructive last, in red. Safe triangle toward submenus (hover intent). Arrows, type-ahead letters, Esc closes one level. Long-press opens the same actions as a bottom sheet on mobile.

**Tooltips**
- ~300ms delay, arrow pointing to trigger, flip at viewport edges, dismiss on mouse leave / Esc / blur / tap outside, max ~300px and one sentence. Never the only place important info lives.

**Command palette (Cmd/Ctrl+K)**
- Fuzzy subsequence matching ("stg" → Settings). Grouped sections (Recent, Actions, Pages). Opens pre-filled with recents/suggestions, never blank. Fully keyboard-driven. Inline spinner for async commands. Nested commands with breadcrumb, Esc goes back one level. Always pair with visible navigation.

---

## 6. Navigation

- Mobile: bottom tabs, 3–5 primary destinations. Desktop: persistent sidebar for 5+ sections. Hamburger only for secondary nav (hiding primary nav costs ~40–56% engagement). Breadcrumbs only for hierarchies > 2 levels.
- **Tabs:** indicator slides (spring, timed with content fade); overflow scrolls horizontally with edge fades (+ chevrons on desktop), never wraps; arrows/Home/End; focus ring color ≠ active color; content fades out → ~80ms → fades in with stable panel height; mobile: segmented control (< 5) or sheet (> 5).
- **Pagination:** cursor-based when rows change often (offset duplicates/skips); numbered for jumping, load-more for on-demand, infinite scroll for feeds; truncate to first, last, current ± 1 with ellipses; page in the URL; restore scroll position on back.
- **Focus:** never `outline: none` without a replacement; ring 2px thick, 2px offset, visible on light and dark; use `:focus-visible`; DOM order = visual order; trap focus in modals and return it to the trigger on close; skip link as first focusable element.

---

## 7. Data

**Tables**
- Sort cycles asc → desc → original. Numbers right-aligned with `font-variant-numeric: tabular-nums`. Sticky header; frozen first column on horizontal scroll (with edge shadow). Density as a token (36/48/60px rows), zebra only at comfortable density. Whole row selectable (tint + accent bar + checkbox).

**Bulk actions**
- Header checkbox has three states (empty / indeterminate / checked); indeterminate resolves to select-all. Name the scope: "Select all 247 matching" and keep the count live as filters change. Selection lives in app state (survives paging), shift-click selects ranges. Destructive bulk actions: run immediately, echo the count, 10s undo.

**Filters & search**
- Filter chips: distinct idle / active (filled + check) / disabled states; OR within a group, AND across groups; result count updates the same frame as the tap; one "Clear all"; overflow as a single scrolling row with edge fade; sticky summary of active filters.
- Search: descriptive placeholder ("Search by name, SKU or brand"); recent searches on focus; autocomplete ranked by popularity with category tags; full keyboard; zero-results screen offers suggestions, spelling alternatives, categories.

**Charts**
- Bar charts start at zero. Chart type from the question (bars compare, lines show change, pies ≤ 5 slices). Aspect ratio so the average slope is near 45°. Remove non-data ink (heavy gridlines, 3D, shadows, boxed legends); label lines directly. One hero color for the series that matters. Title states the takeaway ("Revenue flat since March").

---

## 8. Interactions

- **Hover is an extra, never the only path.** Touch has no hover. Gate hover styles with `@media (hover: hover)`; grow controls with `@media (pointer: coarse)`. Tap targets ≥ 44px even when the icon is 20px.
- **Destructive actions:** verb on the button ("Delete project" / "Keep project"), never in the usual confirm position, red only for destruction, danger zone at page bottom, hold-to-confirm (~300ms) or type-to-confirm for irreversible, grace period (e.g. 14 days) where possible.
- **Inline editing:** hover affordance (pencil or tint); swap text → input with zero layout shift (same font, size, padding; border transparent → accent); Enter saves, Esc cancels, pick one blur rule and keep it; optimistic save with rollback that keeps the draft.
- **Drag and drop:** lift = scale + deeper shadow + slight tilt; show the drop target before release (insertion line vs container highlight); snap to valid slots on structured boards; show valid slots while dragging; ~5s undo after drop.
- **Swipe actions:** hint that they exist; ≤ 2 per direction, consistent directions app-wide; destructive requires partial swipe + tap or full swipe + undo; always an alternative path (long-press menu, button in detail view).
- **Accordion:** header is a `<button>` with `aria-expanded`/`aria-controls`; animate with `grid-template-rows: 0fr → 1fr` (not `height: auto`); chevron on the same curve as the panel; single-open for steps, multi-open for FAQs; keep the tapped header anchored.
- **Realtime collaboration:** interpolate cursors (server ~10Hz → 60fps); color from a hash of user ID; avatar stack with +N; lock selected elements in the editor's color; follow mode.
- **Let CSS own DOM state:** `:has()` for parent styling (`.card:has(:checked)`), layout (`.app:has(aside)`), quantity queries (`.grid:has(> :nth-child(n+5))`) instead of mirrored React state and counting in JS. `isolation: isolate` to contain z-index; z-index needs a positioned element and only competes within its stacking context — no 9999 arms races.

---

## 9. Copy (microcopy)

- Buttons name the reward, not the mechanic: "Create my free account" > "Submit".
- Errors become next steps: "That email's taken — want to log in?" > "Invalid input".
- Write like a helpful colleague; no system-speak ("operation failed", "result set empty").
- Headlines: concrete outcome + audience; no adjective soup (supercharge, seamless, powerful).

---

## 10. Landing pages & the "AI-made" look

**Skeleton (in order):** Hero → Proof → Problem → Solution → CTA.
- Hero answers what / who / why in ~3 seconds: headline + subhead + one CTA. No carousels.
- Proof directly under the hero: real logos, one testimonial with name, role and a measurable result, one hard number.
- Agitate the problem (time, money, frustration) before the solution. Max three benefits framed as outcomes. Same CTA (copy + color) at top and bottom.

**Remove AI tells:**
- Gradient headline full of adjectives → plain statement of outcome and audience.
- Purple blob backgrounds → neutral background; the only color on the product screenshot and primary button.
- Two equal buttons side by side → one primary + a text link with arrow.
- "Trusted by 10,000+ users" + grey logos → one verifiable quote.
- Three icon-in-circle cards (Fast / Secure / Easy) → one product screenshot with three annotations on real UI.
- General rule: replace decoration with the actual product.

**"Premium card" recipe:** generous padding (~40px) and radius (~24px); real type hierarchy (title ~600 weight, description at ~55% opacity); two layered shadows + ~12% hairline border; hover lift ~8px, scale ~1.02, deeper shadow.

---

## Output format (in the conversation, never a file)

Reply in English. Keep each finding to one line; number everything continuously (findings and conflicts share one numbering):

```
## UI audit — <what was reviewed>

Design system detected: <e.g. Tailwind config + CSS variables in globals.css> (or: none)

### P1 – Broken / risky
1. [<location>] <problem> → <fix>
2. ...

### P2 – Feels cheap
3. ...

### P3 – Polish
4. ...

### Conflicts with your design system
5. <topic> — <where it is defined, e.g. `--radius-*` in tokens.css>
   A) Keep existing: <current values> — <why it still works / stays coherent>
   B) Apply skill rule: <new values> — <what changes visually>. Impact: <tokens touched, ~N components/screens>
6. ...

Which fixes do you want me to apply? (all / only P1 / e.g. 1, 3, 4)
And for each conflict, A or B? (e.g. 5A, 6B)
```

Omit any empty section. If there is no design system, replace the conflicts section with one numbered finding proposing a minimal token set (color scale, 4px spacing scale, radius scale, shadow levels, motion durations/easings) and note that other fixes build on it.

If nothing meaningful is wrong, say so in one or two lines instead of inventing findings.