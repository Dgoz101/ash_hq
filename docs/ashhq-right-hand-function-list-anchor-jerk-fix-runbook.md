# ashhq-right-hand-function-list-anchor-jerk-fix — Runbook

Developer workflow for fixing an in-page navigation UX bug on documentation pages in a Phoenix LiveView application. The **immediate task:** right-hand function list (table of contents) — clicking anchors produces a visible jerk / animated nudge when navigating to a section id. This runbook can be followed by a human or an agent (e.g. Cursor Agent) to locate the right-hand function list, reproduce the jerk, diagnose root cause (CSS/JS), propose one or more fixes (without editing repo files unless approved), and verify conceptually. **No repository edits** until the user explicitly types **APPROVE APPLY**.

**Product name:** ashhq-right-hand-function-list-anchor-jerk-fix  
**Target repo:** ash_hq  
**Target area:** Documentation pages where the right-hand function list (table of contents) is rendered.  
**Immediate task:** Right-hand function list — clicking in-page anchors causes visible jerk/nudge; fix so scroll lands once and styling/keyboard/a11y are preserved.

---

## Prerequisites

- Phoenix/LiveView project (ash_hq) at a known repo path (e.g. workspace root).
- Windows environment; Docker + local Phoenix server; Cursor Agent mode (or equivalent).
- Ability to read repo files (LiveView, HEEx, components, CSS, JS). No write access required unless approval is given (see Approval gate).

---

## Approval gate

**Do not edit repo files unless user types APPROVE APPLY.**

All steps in this runbook are **propose-only** until the user explicitly says "APPROVE APPLY." Until then, the agent or human must only locate, reproduce, diagnose, propose (with concrete edits described in text/snippets), and verify conceptually. No file writes, no patches applied.

---

## Constraints (must be respected)

- **No visible layout intent change** — The on-screen appearance of the docs page and TOC must remain the same after any future fix.
- **No break keyboard navigation** — Tab order and focus behavior (Tab → Enter on function links, focus indicator visible) must be preserved.
- **No break a11y semantics** — Screen reader semantics and reading order must be preserved.
- **Propose only** — Do not modify repo code unless the user has typed APPROVE APPLY.
- **Minimal, localized changes** — Prefer changes limited to the function list anchors and/or scroll/highlight behavior; avoid global CSS unless necessary.
- **Explainable and reviewable** — The proposal and verification should be clear enough for a pull request. User may commit and push to their fork; do not open upstream PR.

---

## Workflow (5 steps)

The pipeline has five logical phases, executed by a single actor: **Locate → Reproduce → Diagnose → Propose → Verify**.

---

### Step 1 — Locate

**Goal:** Locate where the right-hand function list is rendered (LiveView, HEEx, components) and identify the anchor markup and classes used for those links.

1. Set the repo path (e.g. workspace root for ash_hq).
2. Identify the docs area:
   - Routes and LiveViews that serve documentation (e.g. doc routes, doc layout).
   - Where the **right-hand function list** (table of contents) is produced — e.g. a post-processor that injects TOC HTML into the doc content, or a component that renders it.
   - The **anchor markup** for TOC links: tag (`<a>`), `href` pattern (`#section-id`), and **CSS/utility classes** on those links.
3. Record:
   - **File path(s)** (e.g. `lib/ash_hq/docs/extensions/render_markdown/post_processors/table_of_contents_generator.ex`, `lib/ash_hq_web/pages/docs.ex`).
   - **Component or template name** and **selector** (e.g. class or structure that identifies the TOC container and TOC links).
   - **Current markup and classes** for the anchors (e.g. `class="text-primary-light-600 dark:text-primary-dark-400 ... block"`).
4. Note the **doc content wrapper** that contains the scroll target (e.g. element with class `nav-anchor` or similar) and any IDs used for in-page targets.

**Output (conceptual):** `location` — filePath, componentName, selector, anchorMarkup, anchorClasses, TOC container description, doc wrapper class/IDs.

---

### Step 2 — Reproduce

**Goal:** Reproduce the jerk locally and document exact reproduction steps.

1. Use the location from Step 1.
2. **Identify one or more docs pages** where the right-hand function list appears and the jerk is visible (e.g. any guide or module doc that has a generated table of contents).
3. **Document exact reproduction steps:**
   - **URL/route** — e.g. `/docs/ash/latest/guide/...` or a specific doc page that shows the TOC.
   - **Viewport widths** — e.g. 1024px, 1280px, 1440px (and optionally mobile) where the jerk is observable.
   - **Click behavior** — e.g. "Click a link in the right-hand Table of Contents; observe a visible jerk or animated nudge as the page scrolls to the target section."
4. Optionally record **browser and OS** (e.g. Chrome on Windows) if the issue is environment-sensitive.

**Output (conceptual):** `reproduction` — pages, URL/route, viewport widths, exact reproduction steps.

---

### Step 3 — Diagnose

**Goal:** Determine root cause by inspecting the CSS and JS involved. Identify the smallest set of classes, rules, or hooks responsible for the jerk.

Use the **root cause diagnosis checklist** below. For each item, inspect the repo and note whether it contributes to the jerk:

1. **Link click styles (TOC anchors)**  
   - Check for **active/focus** styles on the TOC `<a>` elements that use **transform** (e.g. scale, translate), **transition-all**, or other layout-affecting transitions.  
   - If the link (or its parent) has `transition: all` or `transition: transform`, a brief state change on click can cause a visible nudge.

2. **:target styles on the destination element**  
   - Check for **`:target`** CSS rules applied to the destination heading/section (the element whose `id` is the anchor target).  
   - If `:target` applies **transitions** to **layout-affecting properties** (e.g. margin, padding, top, transform), the browser’s scroll + highlight can trigger a transition and cause a jerk.

3. **Scroll behavior applied twice**  
   - Check whether **both** of the following are in effect:  
     - **CSS:** `scroll-behavior: smooth` on `html` (or a scroll container).  
     - **JS:** A LiveView hook or other script that calls **scrollIntoView** (e.g. `behavior: "smooth"`) or a smooth-scroll library when the hash changes.  
   - If the browser already smooth-scrolls via CSS and JS also smooth-scrolls (or vice versa), the scroll can effectively run twice or conflict, producing a jerk.

4. **Smallest set**  
   - After the above, identify the **smallest set** of classes, CSS rules, or JS hooks that, if removed or changed, would eliminate the jerk (e.g. "TOC anchor class with `transition-all`" or "`:target` rule with `transition: margin`" or "duplicate smooth scroll in JS").

**Output (conceptual):** `diagnosis` — rootCause (linkStyles | targetStyles | doubleScroll | combination), culpritSet (smallest set of classes/rules/hooks).

---

### Step 4 — Propose

**Goal:** Propose (but do **not** apply yet) one or more fixes, prioritized by minimal risk.

1. Use the diagnosis from Step 3.
2. **Propose** fix options in this order of preference:
   - **Preferred:** Remove or limit **transforms** and **transition-all** on the function list (TOC) anchors; use **transition-colors** (or similar non-layout transitions) only, so click/focus does not trigger a visible layout nudge.
   - **Alternative:** Adjust **:target** highlight so it does not animate layout — avoid transitions on margin, top, transform; use **background**, **outline**, or **box-shadow** for the highlight instead.
   - **Alternative:** Ensure **only one** scroll mechanism is used — either **CSS** `scroll-behavior: smooth` **or** JS smooth scroll (e.g. LiveView hook / scrollIntoView), not both.
3. Document **tradeoffs** (e.g. changing TOC link styles vs changing :target vs removing one of the scroll mechanisms; impact on hover/focus appearance and keyboard focus visibility).
4. For the preferred option (and optionally others), document:
   - **Files to change** (e.g. table_of_contents_generator.ex, app.css, app.js).
   - **Concrete edits** (snippet-level: what to add, remove, or change).
   - **Rationale** (why this meets constraints and eliminates the jerk).
5. Do **not** edit any repo files unless the user has typed APPROVE APPLY.

**Output (conceptual):** `proposal` — fix options with priority and tradeoffs, file-level changes, snippets, rationale.

---

### Step 5 — Verify

**Goal:** Verify conceptually (no code edits yet) that the proposed fix would: eliminate the jerk, preserve styling and keyboard/screen reader behavior, and avoid regressions.

1. Use the proposal from Step 4.
2. **No jerk:** Confirm that after the fix, clicking a function list anchor would result in a single smooth scroll to the target with no visible nudge/jerk.
3. **Visual styling:** Confirm that hover/focus styling intent would remain (e.g. link color change, focus ring still visible).
4. **Keyboard navigation:** Confirm that Tab → Enter on a function link would still work and the focus indicator would remain visible.
5. **Screen reader semantics:** Confirm that heading structure and link semantics would be unchanged (no removal of landmarks or labels).
6. **Regressions:** Confirm that behavior at common desktop widths (1024, 1280, 1440 px) and on mobile would be unchanged or improved.
7. Fill the **conceptual verification checklist** (see below) and note any caveats or follow-up checks (e.g. "verify in browser after applying").

**Output (conceptual):** `verificationChecklist` — criteria plus passed/notes.

---

## How to reproduce the jerk

Use the outputs from **Step 2 (Reproduce)** to reproduce locally:

1. Start the app (e.g. Docker + local Phoenix server).
2. Open a docs page that shows the **right-hand Table of Contents** (e.g. a guide or module doc with multiple H2/H3 headings). Use the **route** from reproduction (e.g. `/docs/ash/latest/guide/...`).
3. Set browser width to a value where the TOC is visible (e.g. 1024px, 1280px, or 1440px).
4. Click a link in the **right-hand function list** (Table of Contents) that targets a section lower or higher on the page.
5. Observe: a visible **jerk** or **animated nudge** as the page scrolls to the target (e.g. content shifts briefly, or scroll appears to "double" or correct itself).

Record the exact route, width(s), and click behavior so the fix can be validated against the same scenario.

---

## Root cause diagnosis checklist (what to inspect in CSS/JS)

When performing **Step 3 — Diagnose**, inspect the following and note which apply:

| Check | What to inspect | Notes |
|-------|-----------------|--------|
| **Link click styles** | TOC anchor classes and any CSS for `a:active`, `a:focus`, or parent transitions. Look for `transition`, `transition-all`, `transform`, `scale`, `translate`. | If TOC links (or their container) have `transition-all` or transform on focus/active, the jerk may be from the link’s own transition. |
| **:target styles** | Global or prose CSS for `:target` (e.g. on headings or sections). Look for `transition` on `margin`, `padding`, `top`, `transform`. | If the destination element transitions layout when it becomes `:target`, the jerk may be from that. |
| **Scroll behavior** | `app.css`: `scroll-behavior: smooth` on `html`. `app.js`: `scrollIntoView({ behavior: "smooth" })`, or smooth-scroll library (e.g. `smooth-scroll-into-view-if-needed`) used on hash change or link click. | If both CSS smooth scroll and JS smooth scroll are active for the same navigation, scroll can run twice or conflict. |
| **Smallest set** | After the above, list the minimal classes, rules, or hooks that cause the jerk (e.g. "TOC anchor has no transition; :target has transition on margin; remove :target transition" or "remove JS scrollIntoView for hash links"). | Use this to choose the minimal fix in Step 4. |

---

## Proposed fix options with tradeoffs

Use the **Step 4 (Propose)** output. In the runbook or proposal, include:

- **Option A (Preferred):** Remove/limit transforms and `transition-all` on the function list anchors; use `transition-colors` (or similar) only. *Tradeoffs:* Minimal change to TOC links; hover/focus color change can stay; no change to :target or scroll mechanism.
- **Option B:** Adjust `:target` highlight to avoid animating layout — no transitions on margin/top/transform; use background/outline/box-shadow. *Tradeoffs:* Fixes jerk from destination transition; may require ensuring no other layout transition on :target.
- **Option C:** Use only one scroll mechanism — either CSS `scroll-behavior: smooth` or JS smooth scroll, not both. *Tradeoffs:* Removes double-scroll conflict; need to ensure one path is used for in-page hash links and that behavior is consistent (e.g. no regression on other hash navigation).

Recommendation: State which option is preferred and why (e.g. Option A for minimal risk and localized change). Include **snippets** (before/after or key class/rule changes) for the chosen option.

---

## Conceptual verification checklist

Before any code change, verify conceptually that the proposal satisfies:

| Criterion | Passed | Notes |
|-----------|--------|-------|
| Click on function anchors no longer causes nudge; scroll lands once | ☐ | |
| Visual styling intent preserved (hover/focus still clear) | ☐ | |
| Keyboard: Tab → Enter on function link works; focus indicator visible | ☐ | |
| Screen reader semantics unchanged | ☐ | |
| No regressions at 1024, 1280, 1440 px; mobile behavior unchanged | ☐ | |

Fill this table (or equivalent) as part of **Step 5 — Verify** and attach any caveats or follow-up checks (e.g. "confirm in browser after applying").

---

## Future: applying the fix (after approval)

**This section describes post-approval behavior only. No repository edits are made in the current phase.**

Once you have reviewed the proposal and verification checklist and are ready to apply the fix:

1. **Send an explicit instruction:** Type **APPROVE APPLY** (or equivalent) to authorize the agent or human to edit repository files.
2. **Apply the approved fix:** Implement the chosen fix option in the files identified in the proposal. Make only the edits described in the proposal (snippets, classes, or rules). Do not change visible layout intent; do not break keyboard navigation or a11y semantics.
3. **Re-verify in the environment:** In a browser at common desktop widths (e.g. 1024, 1280, 1440 px) and optionally on mobile:
   - Click function list anchors; confirm no jerk; scroll lands once.
   - Confirm hover/focus styling is still clear.
   - Confirm Tab → Enter on function links works and focus indicator is visible.
   - Confirm screen reader semantics are unchanged (e.g. test with accessibility tools if available).
4. **Optional:** Run any existing tests (e.g. docs or layout tests) and fix any regressions.
5. **Review as a pull request:** Ensure changes are explainable and reviewable; add a short PR description referencing this runbook and the chosen fix option. User may commit and push to their fork; do not open upstream PR.

Until you send **APPROVE APPLY**, the agent or human must **not** modify any repository files.

---

## Message flow (summary)

```
Invoker → AnchorJerkAnalyst (Locate → Reproduce → Diagnose → Propose → Verify) → Invoker
```

---

## Artifact paths

- **Tool spec (machine-readable):** `tools/ashhq-right-hand-function-list-anchor-jerk-fix.jsonld`
- **Runbook (this file):** `docs/ashhq-right-hand-function-list-anchor-jerk-fix-runbook.md`

---

## References

- Target repo: ash_hq (Phoenix LiveView application).
- Target area: Documentation pages where the right-hand function list (table of contents) is rendered.
- Goal: Eliminate visible jerk/nudge when clicking in-page anchors in the right-hand function list, while preserving layout intent, keyboard navigation, and accessibility semantics. Apply behavior: PROPOSE ONLY until APPROVE APPLY.

Created using AALang and Gab
