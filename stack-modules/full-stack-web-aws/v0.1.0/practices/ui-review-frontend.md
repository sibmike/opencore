# UI Review: Next.js + Tailwind Responsive Audit

Run this audit before every commit that touches a page, layout component, or global
stylesheet. Visual regressions are invisible in diffs — only a browser viewport reveals
them.

---

## When to run this audit

Run the full protocol for any commit that touches:

- A marketing or landing page.
- A presentation/slide deck component.
- A screenshot or mock page.
- A shared layout, nav bar, or sidebar.
- Any Tailwind class that controls width, font size, grid columns, or overflow.

Skip it only for commits that are purely server-side (API routes, service logic,
database queries) with no rendered output.

---

## Protocol overview

1. Start the dev server.
2. Audit at desktop (1280×900) — screenshot every section.
3. Audit at mobile (375×812) — screenshot every section.
4. Inventory every issue found before fixing any of them.
5. Apply fixes using the cheat sheet.
6. Re-verify both viewports.
7. Run `npx tsc --noEmit`.
8. Commit only after all steps pass.

Collect first, fix second. Fixing mid-audit causes you to miss issues downstream.

---

## Step 1 — Start the dev server

```
preview_start frontend
```

Confirm the server is live before resizing. A stale build will produce misleading
screenshots.

---

## Step 2 — Desktop audit (1280×900)

```js
preview_resize { width: 1280, height: 900 }
```

- Navigate to each affected page.
- For scrollable pages: scroll through every section and screenshot each one.
- For scroll-snap decks: iterate each slide via `.deck-slide` `offsetTop` and
  screenshot at each position.

---

## Step 3 — Mobile audit (375×812)

```js
preview_resize { preset: "mobile" }  // 375×812
```

Same process: every section, every slide. Screenshot each one.

---

## Step 4 — Per-screenshot checklist

For every screenshot taken at both viewports, verify:

- [ ] No orphaned word (single word alone on the last line of a heading or paragraph).
- [ ] No mid-word hyphenation (`"AI-\npowered"`, `"feature-\nready"`).
- [ ] No mid-phrase break that separates logically-bound words.
- [ ] No text clipped at the right edge.
- [ ] No body text below `text-base` (16 px) minimum.
- [ ] Grids stack properly on mobile — no 3-column or wider layout at 375 px.
- [ ] Fixed-width elements fit within the viewport.
- [ ] Images and screenshots render at their full intended resolution.

---

## Step 5 — 10 bad patterns and their fixes

### Pattern 1 — Orphaned word at end of heading

**Symptom:** The last word of a heading wraps alone onto its own line.

**Why it matters:** Single orphan words read as broken copy and are invisible in code review.

**Fix (pick one):**
- Insert `&nbsp;` between the last two words of the heading.
- Add `<br className="sm:hidden" />` earlier in the heading to force a cleaner mobile break.
- Step the heading down: `text-2xl sm:text-4xl` so the full phrase fits on one line at 375 px.

---

### Pattern 2 — Mid-word hyphenation at large sizes

**Symptom:** Tailwind's default word-break splits a hyphenated compound across lines on
narrow viewports (`"Pre-`  `/`  `cleared"`).

**Why it matters:** CSS hyphenation is off by default; the break happens at the literal
hyphen character when the word is too wide.

**Fix:**
- Reduce the mobile font size: `text-3xl sm:text-6xl` instead of `text-5xl sm:text-6xl`.
- Replace the hyphen with a non-breaking hyphen: `Pre&#8209;cleared`, `feature&#8209;ready`.
- Use `&nbsp;` to bind short compound words: `AI&nbsp;Score`.

---

### Pattern 3 — Unconditional `<br />` breaks on desktop

**Symptom:** A `<br />` added to fix a mobile line break causes an awkward forced break on
desktop where the text would have flowed naturally.

**Why it matters:** Desktop viewports are wide enough for the phrase to fit on one line; the
`<br />` overrides natural flow.

**Fix:** Replace every unconditional `<br />` with `<br className="sm:hidden" />`. Before
committing, ask: "Do I want this break at 1280 px too?" If the answer is no, scope it to
mobile.

---

### Pattern 4 — Grid forced to 3+ columns on mobile

**Symptom:** `grid grid-cols-3` renders three cramped, unreadable columns at 375 px.

**Why it matters:** Content at ~125 px column width is illegible and items may overflow
into each other.

**Fix:** Always start at one column and widen at a breakpoint:

```html
<!-- Wrong -->
<div class="grid grid-cols-3">

<!-- Right -->
<div class="grid grid-cols-1 sm:grid-cols-3">
<!-- or -->
<div class="grid grid-cols-1 md:grid-cols-3">
```

---

### Pattern 5 — Fixed widths that overflow on mobile

**Symptom:** A `w-72` (288 px) element overflows a 375 px viewport once padding is
accounted for.

**Why it matters:** `288 px + 2 × 20 px padding = 328 px`, which fits — but `w-80`
(320 px) does not, and many elements also have border or margin.

**Fix:**

```html
<!-- Wrong -->
<div class="w-72">

<!-- Right: fluid on mobile, fixed on desktop -->
<div class="w-full sm:w-72">

<!-- Right: centered, max-constrained -->
<div class="max-w-xs mx-auto">
```

---

### Pattern 6 — Horizontal overflow from absolute-positioned elements

**Symptom:** A decorative element with `absolute left-1/4 w-96` extends beyond the
viewport, causing a horizontal scrollbar and clipping text on the right.

**Why it matters:** `overflow-x-auto` on the page body does not prevent the layout from
widening; it adds a scrollbar instead.

**Fix:** Add `overflow-x-hidden` to the root page `<div>` (the topmost element of the page
component, not a section or article inside it).

**Debug technique — find the offending element:**

```js
preview_eval: (function () {
  const all = document.querySelectorAll('*');
  let maxRight = 0; let offender = '';
  for (const el of all) {
    const r = el.getBoundingClientRect();
    if (r.right > maxRight) {
      maxRight = r.right;
      offender = el.tagName + '.' + el.className.substring(0, 50);
    }
  }
  return JSON.stringify({ maxRight, offender, viewportWidth: window.innerWidth });
})()
```

---

### Pattern 7 — Text too small for presentation or marketing pages

**Symptom:** `text-xs` (12 px) or `text-[10px]` used for body copy, descriptions, or
slide content.

**Why it matters:** Small text fails readability at arm's length and fails accessibility
contrast thresholds at small sizes.

**Fix:**
- `text-base` (16 px) is the minimum for all body content.
- `text-sm` (14 px) is acceptable only for tertiary labels, timestamps, and footnotes.
- Use `text-lg` or `text-xl` for slide or projected content.

---

### Pattern 8 — Table that doesn't fit on mobile

**Symptom:** A multi-column table (3+ columns) gets cut off or scrolls horizontally at
375 px. `overflow-x-auto` does not help when the table is inside a viewport-height
scroll-snap slide.

**Why it matters:** Horizontal scroll inside a vertically-constrained slide is confusing
and effectively hides content.

**Fix:** Dual layout — card stack on mobile, table on desktop:

```jsx
{/* Mobile: card layout */}
<div className="sm:hidden space-y-3">
  {rows.map(row => (
    <div key={row.id} className="bg-card rounded-lg p-4">
      <p className="font-medium">{row.label}</p>
      <p className="text-sm text-muted-foreground">{row.value}</p>
    </div>
  ))}
</div>

{/* Desktop: full table */}
<div className="hidden sm:block">
  <table className="w-full">
    {/* ... */}
  </table>
</div>
```

---

### Pattern 9 — Heading using `text-5xl` or larger on mobile

**Symptom:** A heading sized `text-5xl` (48 px) or `text-6xl` (60 px) on mobile forces
ugly word breaks because 375 px minus padding is too narrow for the glyph widths.

**Why it matters:** The break happens at word boundaries chosen by the browser, not at
semantically clean points.

**Fix:** Step down one or two sizes for mobile:

```html
<!-- Wrong -->
<h1 class="text-5xl">

<!-- Right -->
<h1 class="text-3xl sm:text-5xl">
<h1 class="text-4xl sm:text-6xl">
```

---

### Pattern 10 — `overflow-hidden` clipping legitimate content

**Symptom:** A section with `overflow-hidden` (added to contain decorative background
orbs) also clips text that extends slightly beyond the element due to font rendering,
letter-spacing, or drop-shadow.

**Why it matters:** Scoping `overflow-hidden` to a section clips its own children.
Decorative elements are not children of the section they decorate — they are positioned
relative to the page.

**Fix:** Move `overflow-x-hidden` up to the page root `<div>`. The section loses its
overflow containment, but the page root catches any absolute-positioned decoration that
would extend past the viewport.

```jsx
// Wrong: clips section children
<section className="overflow-hidden relative">
  <div className="absolute -left-20 w-96 h-96 bg-violet-500/10 rounded-full" />
  <h2>Heading that gets clipped</h2>
</section>

// Right: contains decoration at page level, frees section children
<div className="overflow-x-hidden">          {/* page root */}
  <section className="relative">
    <div className="absolute -left-20 w-96 h-96 bg-violet-500/10 rounded-full" />
    <h2>Heading renders fully</h2>
  </section>
</div>
```

---

## Fix techniques cheat sheet

| Need | Technique |
|------|-----------|
| Keep two words together | `&nbsp;` between them |
| Keep a hyphenated compound together | `&#8209;` (non-breaking hyphen) |
| Line break on mobile only | `<br className="sm:hidden" />` |
| Line break on desktop only | `<br className="hidden sm:inline" />` |
| Prevent horizontal overflow | `overflow-x-hidden` on root `<div>` |
| Smaller heading on mobile | `text-3xl sm:text-5xl` |
| Stack cards on mobile | `grid-cols-1 sm:grid-cols-3` |
| Full-width mobile, fixed desktop | `w-full sm:w-72` |
| Hide decoration on mobile | `hidden md:block` |
| Alternate mobile layout | `sm:hidden` + `hidden sm:block` dual markup |

---

## Mobile breakpoint reference (375 px)

Check these at 375×812 before every commit:

- Nav bar usable? Items hidden (`hidden md:flex`) or collapsed into a hamburger?
- Headings readable without mid-word breaks?
- Multi-column grids stacked to a single column?
- Fixed-width elements constrained to viewport width?
- Scroll-snap deck side navigation out of the way? (It should be `hidden sm:flex`.)
- Absolute-positioned decorations not causing horizontal overflow?
- No body text smaller than `text-base`?

---

## Desktop breakpoint reference (1280 px)

Check these at 1280×900 before every commit:

- No unconditional `<br />` causing awkward forced breaks?
- No `&nbsp;` clusters forcing unnatural spacing or over-long unbroken phrases?
- Headings that look fine on mobile still proportional at full width?
- Multi-column layouts using the full available width?
- Images and screenshots at their full intended resolution?
- Card grids filling columns (not stretched into one-column layout at desktop)?

---

## Pre-commit verification script

Mandatory for any commit touching marketing pages, presentation decks, screenshot pages,
or shared layout components.

**Step 1 — Dev server**
```
preview_start frontend
```
Confirm the server responds before proceeding.

**Step 2 — Desktop audit**
```js
preview_resize { width: 1280, height: 900 }
```
Navigate each affected page. Screenshot every section and every slide.

**Step 3 — Mobile audit**
```js
preview_resize { preset: "mobile" }
```
Same pages, same sections, same slides. Screenshot each.

**Step 4 — Issue inventory**
Write down every orphan, clip, overflow, and bad break you see. Do not fix anything yet.
Collect the full list.

**Step 5 — Apply fixes**
Work through the issue list using the cheat sheet. One fix at a time when the scope is
uncertain — it is easy to introduce a desktop regression while fixing a mobile issue.

**Step 6 — Re-verify**
Screenshot the affected sections again on both viewports. Confirm each issue is resolved
and no new issue appeared.

**Step 7 — TypeScript check**
```bash
npx tsc --noEmit
```
Fix any type errors before proceeding.

**Step 8 — Commit**
Only after all seven steps above pass.

---

## Layout patterns worth knowing

### Always-mounted panel for stateful tabs

Use `display:none` toggling (`hidden` / `block`) rather than conditional rendering or
`translate-x-full` transforms when a panel must preserve internal state across tab
switches (e.g., a chat widget, a form in progress).

```jsx
// Correct: panel stays mounted, state is preserved
<div className={activeTab === 'chat' ? 'block' : 'hidden'}>
  <ChatWidget />
</div>

// Wrong: unmounts on tab switch, state is lost
{activeTab === 'chat' && <ChatWidget />}

// Also wrong: CSS transform moves the visual box but not the hit area
<div className={activeTab === 'chat' ? '' : 'translate-x-full overflow-hidden'}>
  <ChatWidget />
</div>
```

### Responsive gaps as a gap list, not individual fixes

When auditing a page for the first time, walk through every section before touching
anything. Common gaps in a Next.js + Tailwind SaaS app:

- No responsive mobile layout (three breakpoints, collapsible sidebar).
- Document preview supports one MIME type only (PDF); others need separate strategies.
- Missing download affordance on shared document views.

Treat these as a prioritized list, not individual one-off fixes.

---

## Cross-references

- `core/practices/anti-patterns.md` — generic anti-patterns (framework-agnostic).
- `core/documentation/checklist-protocol.md` — checklist discipline and cardinal rule.
- `practices/conventions-fullstack.md` — TypeScript and Tailwind coding conventions.
- `practices/testing-fullstack.md` — Playwright screenshot pipeline (mock pages, crop parameters).

---

<!-- PROVENANCE
stack: full-stack-web-aws v0.1.0
bootstrap source: dreams_v1 project docs (single-project extraction)
fragments consumed: F-060, F-064, F-137, F-138, F-139, F-140, F-141, F-142, F-143,
  F-144, F-145, F-146, F-147, F-148, F-149, F-150, F-151, F-152, F-153, F-154, F-161
fragment sources: ui-review-checklist.md (18), functionality-audit.md (2), workflows.md (1)
yaml reads: split-reports/stack-extraction/manifest/per-doc/ui-review-checklist.yaml
generated: 2026-05-10
-->
