# ashhq-docs-scroll-container-usability-fix — Runbook

Developer workflow for fixing a scrolling usability issue on documentation pages in a Phoenix LiveView application. The **immediate task:** docs content area scrolls inside a narrow container, making scrolling unnecessarily difficult because users must scroll within the content panel instead of scrolling anywhere on the page. This runbook can be followed by a human or an agent (e.g. Cursor Agent) to analyze layout and scroll behavior, locate the DOM/CSS responsible for constrained scrolling, reproduce the narrow-scroll behavior, propose one or more fixes (without editing repo files unless approved), and verify conceptually. **No repository edits** until the user explicitly types **APPROVE APPLY**.

**Product name:** ashhq-docs-scroll-container-usability-fix  
**Target repo:** ash_hq  
**Target area:** Documentation pages where main content scrolling is constrained.  
**Immediate task:** Identify why scrolling is limited to a narrow content area; propose a fix so users can scroll from anywhere on the page (or a widened scroll area) while preserving layout, sticky sidebars, keyboard navigation, and accessibility.

---

## Prerequisites

- Phoenix/LiveView project (ash_hq) at a known repo path (e.g. workspace root).
- Windows environment; Docker + local Phoenix server; Cursor Agent mode (or equivalent).
- Ability to read repo files (LiveView, HEEx, components, Tailwind/CSS). No write access required unless approval is given (see Approval gate).

---

## Approval gate

**Do not edit repo files unless user types APPROVE APPLY.**

All steps in this runbook are **propose-only** until the user explicitly says "APPROVE APPLY." Until then, the agent or human must only analyze, locate, reproduce, propose (with concrete edits described in text/snippets), and verify conceptually. No file writes, no patches applied.

---

## Constraints (must be respected)

- **Do not significantly change visible layout intent** — The on-screen appearance of the docs page (sidebars, headers, content placement) must remain the same after any future fix.
- **Do not break keyboard navigation** — Tab order and focus behavior must be preserved.
- **Do not break a11y semantics** — Screen reader semantics and reading order must be preserved.
- **Propose only** — Do not modify repo code unless the user has typed APPROVE APPLY.
- **Minimal, localized changes** — Prefer changes limited to the scroll container and layout that creates it; avoid global CSS unless clearly justified.
- **Explainable and reviewable** — The proposal and verification should be clear enough for a pull request. User may commit and push to their fork; do not open upstream PR.

---

## Fast triage (browser DevTools)

Use this to **quickly identify the active scroll container** before or alongside the full workflow. No repo edits; inspection only.

1. **Open a long docs page** in the browser (e.g. `/docs/ash/latest/guide/...`) and open DevTools (F12).
2. **Find which element actually scrolls:**
   - **Chrome/Edge:** Right-click the main content area → Inspect. In the Elements panel, select the node that contains the doc content. Use **Ctrl+F** in the Styles panel and search for `overflow`. Walk up the DOM: the first ancestor that has `overflow: auto`, `overflow-y: auto`, or `overflow: scroll` and a **constrained height** (e.g. `height`, `max-height`, or flex/grid that gives it a fixed viewport height) is likely the **active scroll container**.
   - **Firefox:** Inspect the content area; in the Rules panel look for `overflow` and height on the selected element and its ancestors. The element whose scrollbar moves when you scroll the content is the scroll container.
3. **Confirm constrained height:** For that element (and its parent chain), check Computed or Styles for: `height` / `max-height` (e.g. `100vh`, `100%`, pixel values) or flex/grid (e.g. `flex: 1` with a parent that has a fixed height). The combination of **overflow: auto/scroll** + **constrained height** on the same element (or a flex child with `min-h-0` + overflow) is what creates an isolated scroll context.
4. **Single main scroll context vs narrow inner:** If the **document** (or `html`/`body`) has no overflow and does not scroll, but an inner div does, you have a **narrow inner scroll container**. After a fix, you want a **single main scroll context** (typically document/body or one full-width wrapper) so that scrolling with the pointer anywhere on the page moves that context; sticky sidebars then stick within that same scroll context.

**Outcome:** Note the scroll container’s tag, class names, and approximate DOM path. Use this in **Step 2 (Locate)** to search the codebase for the same patterns.

---

## Workflow (5 steps)

The pipeline has five logical phases, executed by a single actor: **Analyze → Locate → Reproduce → Propose → Verify**.

---

### Step 1 — Analyze

**Goal:** Analyze the docs page layout and scroll behavior: which container(s) are scrollable, why scrolling is constrained to a narrow content area, and how main content, sidebar(s), and page shell are structured.

Optionally, use the **Fast triage** section above (before or alongside this step) to identify the active scroll container in the browser.

1. Set the repo path (e.g. workspace root for ash_hq).
2. **Identify the docs area:**
   - Routes and LiveViews that serve documentation (e.g. doc routes, doc layout).
   - The **page shell**: outer wrapper, main content region, left nav (sidebar), right function list (if present).
   - How these are composed (LiveView, HEEx, components, Tailwind).
3. **Identify scroll behavior:**
   - Which container(s) have scroll (e.g. `overflow-y: auto` or `overflow-y: scroll`).
   - Whether the **full page** scrolls or only an **inner container** (e.g. content column).
   - Why scrolling is constrained (preliminary: e.g. "content is inside a fixed-height or flex child that scrolls").
4. Produce a **layout summary**: which modules/templates/components render the shell, main content, and sidebars; how they are composed; which element(s) are the scroll container(s).

**Output (conceptual):** `layoutSummary` — scrollable container(s), main content vs sidebars vs shell, LiveView/HEEx/components/Tailwind structure.

---

### Step 2 — Locate

**Goal:** Locate the specific DOM element(s) and CSS rules responsible for the constrained scrolling.

1. Use the layout summary from Step 1 (and, if used, the **Fast triage** outcome from the section above).
2. **Search the codebase for these CSS/Tailwind patterns** (they commonly create or constrain scroll):
   - **Overflow (creates scroll container):** `overflow-y-auto`, `overflow-y-scroll`, `overflow-auto`, `overflow-scroll`, or raw CSS `overflow: auto` / `overflow-y: auto`.
   - **Constrained height (required for inner scroll):** `h-screen`, `h-full`, `min-h-screen`, `min-h-0`, `max-h-screen`, `max-h-full`, or raw `height: 100vh`, `height: 100%`, `min-height: 0`, `max-height: …`.
   - **Flex/grid (often parents of scroll container):** `flex-1`, `flex-grow`, `min-h-0` (critical for flex children to shrink and scroll), `grid` + fixed row height, or a parent with `h-screen` / `100vh` that forces children to a viewport-sized box.
   - **Sticky (context matters):** `sticky`, `top-*` — note whether the sticky element is **inside** the scroll container (sticky within the narrow column) or **outside** (sticky within document scroll).
3. **Typical file locations to check:**
   - **Docs layout / page shell:** Root layout or app layout that wraps the docs (e.g. `lib/ash_hq_web/templates/layout/…`, or a docs-specific layout).
   - **Docs page / LiveView:** The LiveView or layout that renders the docs page structure (e.g. `lib/ash_hq_web/pages/docs.ex`, or a component that wraps the main content + sidebars).
   - **Components:** Any component that wraps the main content column or the “content + sidebars” region (e.g. a docs content wrapper, sidebar layout component).
   - **Tailwind/CSS:** Custom CSS or Tailwind @apply that sets overflow or height (e.g. `assets/css/app.css`, or component-scoped styles).
4. **Identify and record:**
   - **overflow rules** — Which elements have the overflow classes/rules above, and in which file.
   - **Fixed or absolute heights** — Which elements use the height patterns above and constrain the scroll region.
   - **Flex/grid layouts** — Parent chain (e.g. `flex flex-col h-screen` → `flex-1 min-h-0 overflow-y-auto` on the content child).
   - **Sticky** — Whether sidebars/headers are inside or outside the scroll container; note their scroll context.
5. **Record:** File path(s), selectors (class names, structure) for the scroll container and its parents, and current markup/classes (e.g. `class="flex-1 min-h-0 overflow-y-auto"`).

**Output (conceptual):** `rootCauseLocation` — DOM elements, overflow rules, fixed/absolute heights, flex/grid creating scroll containers, sticky interaction; file paths, selectors, current markup/classes.

---

### Step 3 — Reproduce

**Goal:** Reproduce the narrow-scroll behavior locally and document exact reproduction steps.

1. Use the location from Step 2.
2. **Identify one or more docs pages** where the narrow scroll area is most noticeable (e.g. a long guide or module doc).
3. **Document exact reproduction steps:**
   - **URL/route** — e.g. `/docs/ash/latest/guide/...` or a specific doc page.
   - **Viewport sizes** — e.g. 1024px, 1280px, 1440px (and optionally mobile) where the constraint is observable.
   - **Expected vs actual behavior** (be explicit about scroll context):
     - **Expected (single main scroll context):** There is a single main scroll context (e.g. the document/body or one full-width wrapper). The user can scroll the main content by using the mouse wheel or trackpad **anywhere** on the page — over the left sidebar, over the content column, or over the right function list. Sticky sidebars (if any) stick within that same scroll context; they do not create a separate scroll. Keyboard (Page Up/Down, Space) scrolls the same context.
     - **Actual (narrow inner scroll container):** The main scroll context is **not** the document/body but a **narrow inner container** (e.g. only the content column). Scrolling with the pointer over the left sidebar or right function list does **not** move the main content (or scrolls nothing); only when the pointer is over that inner container does scrolling work. Sticky sidebars may sit outside this scroll container, so they stay fixed while only the content column scrolls — resulting in a small scroll “hit area” and a worse UX.
4. Optionally record **browser and OS** if the issue is environment-sensitive.

**Output (conceptual):** `reproduction` — pages, URL/route, viewport sizes, expected vs actual scroll behavior, exact reproduction steps.

---

### Step 4 — Propose

**Goal:** Propose (but do **not** apply yet) one or more fixes that improve scroll usability while preserving layout intent and sidebar behavior.

1. Use the root cause from Step 2 and reproduction from Step 3.
2. **Propose** fix options in this order of preference:
   - **Preferred:** Allow scrolling on the **main page/body** so users can scroll anywhere (e.g. move scroll to the viewport or a single outer container; remove or relax the inner scroll container that limits scroll to the content column).
   - **Alternative:** **Widen** the effective scroll area so scrolling is not limited to a narrow column (e.g. make the scroll container span content + one sidebar, or restructure so the "scrollable" region is larger).
3. **Constraints for any fix:**
   - Sidebars (left nav, right function list) should continue to behave as intended (e.g. sticky if needed), but must **not** force an isolated narrow scroll container.
   - Avoid introducing **nested scroll containers** unless absolutely necessary.
   - Preserve visible layout intent; no regression to keyboard navigation or a11y semantics.
4. Document **tradeoffs** (e.g. full-page scroll vs widened scroll; impact on sticky behavior, print, or mobile).
5. For the preferred option (and optionally others), document:
   - **Files to change** (e.g. layout template, docs page component, Tailwind classes).
   - **Concrete edits** (snippet-level: what to add, remove, or change).
   - **Rationale** (why this meets constraints and improves scroll usability).
6. Do **not** edit any repo files unless the user has typed APPROVE APPLY.

**Output (conceptual):** `proposal` — fix options with tradeoffs, preferred and alternatives, file-level changes, snippets, rationale.

---

### Step 5 — Verify

**Goal:** Verify conceptually (no code edits yet) that the proposed fix would: establish a single main scroll context (or widened scroll), preserve layout and sticky behavior, and avoid regressions. Use explicit checks that **prove the issue is resolved**. All checks are PROPOSE-ONLY; no repo edits until APPROVE APPLY.

1. Use the proposal from Step 4.
2. **Single main scroll context (issue resolved):**
   - Confirm that after the fix there would be a **single main scroll context** — either the document (e.g. `html`/`body` scrolling) or one full-width wrapper that contains the main content and is the only scroll container. No narrow inner column should be the sole scroll container for the main content.
   - Confirm that scrolling with the pointer **anywhere** on the page (over left sidebar, content, right function list) would move that same scroll context, so the user does not have to “hit” a narrow content column to scroll.
3. **Sticky sidebars still correct:**
   - Confirm that left nav and right function list (if sticky) would still **stick as intended** within the (now single) scroll context — i.e. they remain in view while the user scrolls the main content, without creating a separate scroll container or breaking layout.
4. **Layout:** Confirm no regressions to layout or content visibility (sidebars and headers still visible and positioned as intended).
5. **Keyboard navigation:** Confirm that Tab order and focus behavior would remain intact; Page Up/Down / Space would scroll the main scroll context.
6. **Accessibility:** Confirm that semantics and reading order would remain intact.
7. **Viewports:** Confirm behavior would be consistent across common desktop widths (1024, 1280, 1440 px) and unchanged on mobile unless intentionally improved.
8. Fill the **conceptual verification checklist** (see below), including the explicit checks for single scroll context and sticky behavior. Note any caveats or follow-up checks (e.g. "verify in browser after applying").

**Output (conceptual):** `verificationChecklist` — criteria plus passed/notes.

---

## How to reproduce the narrow-scroll behavior

Use the outputs from **Step 3 (Reproduce)** to reproduce locally:

1. Start the app (e.g. Docker + local Phoenix server).
2. Open a **docs page** where the main content is long enough to scroll (e.g. a guide or module doc). Use the **route** from reproduction (e.g. `/docs/ash/latest/guide/...`).
3. Set browser width to a value where the layout is visible (e.g. 1024px, 1280px, or 1440px).
4. **Expected (single main scroll context):** A single main scroll context (document/body or one full-width wrapper). Scrolling with mouse wheel or trackpad **anywhere** on the page — over the left sidebar, content column, or right function list — moves the main content. Sticky sidebars stick within that same context; they do not define a separate scroll.
5. **Actual (narrow inner scroll container):** The main content scrolls only inside a **narrow inner container** (e.g. the content column). When the pointer is over the left sidebar or right function list, scrolling does not move the main content (or does nothing). Only when the pointer is over that inner container does scrolling work — so the scroll “hit area” is unnecessarily small.
6. Optionally: try keyboard (Page Up/Down, Space) and note whether focus/scroll behavior is also constrained.

Record the exact route, width(s), and scroll behavior so the fix can be validated against the same scenario.

---

## Root cause diagnosis (what creates the isolated scroll container)

When performing **Step 2 (Locate)** and **Step 4 (Propose)**, the following are typical causes of constrained scrolling:

| Cause | What to look for | Notes |
|-------|------------------|--------|
| **overflow on content only** | A single column (main content) has `overflow-y: auto` or `overflow-y: scroll` while the page shell does not scroll. | Scroll is limited to that element; body or outer wrapper does not scroll. |
| **Fixed/absolute height** | The scroll container (or a parent) has a fixed height (e.g. `h-screen`, `100vh`, `max-h-*`). | Creates a viewport-sized box; only its overflow scrolls. |
| **Flex/grid creating scroll** | Parent is flex/grid with `flex-1` or `min-h-0` and a child has `overflow-auto`. | Classic "flex child scroll" pattern; only that child scrolls. |
| **Sticky + scroll** | Sidebars are sticky inside a scroll container, or the scroll container is built to accommodate sticky headers. | Sticky is fine; the key is whether the **scroll context** is the full page or only the content column. |

**Root cause summary:** Identify the **smallest set** of elements and rules that, if changed, would allow a single main scroll context (or widened scroll) without breaking layout or sticky behavior.

---

## Proposed fix options with tradeoffs

Use the **Step 4 (Propose)** output. In the runbook or proposal, include:

- **Option A (Preferred):** Allow scrolling on the **main page/body** so users can scroll anywhere. Move scroll to the viewport or a single outer wrapper; remove or relax the inner scroll container so the content participates in page scroll. *Tradeoffs:* Best UX (scroll anywhere); may require adjusting sticky sidebars to work with document scroll (e.g. `position: sticky` on viewport scroll); minimal nested scroll.
- **Option B:** **Widen** the effective scroll area so the scroll container is not a narrow column (e.g. make one scroll region that includes content + one sidebar, or restructure so the scrollable region is the main area). *Tradeoffs:* Improves usability without full-page scroll; may require more layout changes; ensure no new nested scroll containers.
- **Option C (fallback):** If full-page or widened scroll is not feasible without breaking layout, document the constraint and suggest a **minimal** change (e.g. ensure the scroll container has a larger hit area or is clearly indicated) and any follow-up work.

Recommendation: State which option is preferred and why. Include **snippets** (before/after or key class/structure changes) for the chosen option. Ensure sidebars remain sticky as intended and keyboard/a11y are preserved.

---

## Conceptual verification checklist

Before any code change, verify conceptually that the proposal satisfies these checks (use them in **Step 5 — Verify** to prove the issue is resolved):

| Criterion | Passed | Notes |
|-----------|--------|-------|
| **Single main scroll context:** There is a single main scroll context (document/body or one full-width wrapper); no narrow inner column is the sole scroll container for main content. | ☐ | |
| **Scroll from anywhere:** User can scroll the main content by using mouse wheel/trackpad anywhere on the page (sidebar, content, right function list), not only over a narrow content column. | ☐ | |
| **Sticky sidebars still correct:** Left nav and right function list (if sticky) still stick as intended within the single scroll context; no separate scroll container; layout unchanged. | ☐ | |
| No regressions to layout or content visibility | ☐ | |
| Keyboard navigation and accessibility semantics remain intact | ☐ | |
| Behavior consistent across 1024, 1280, 1440 px; mobile unchanged or improved | ☐ | |

Fill this table (or equivalent) as part of **Step 5 — Verify** and attach any caveats or follow-up checks (e.g. "confirm in browser after applying"). All checks are conceptual only; no repo edits until APPROVE APPLY.

---

## Future: applying the fix (after approval)

**This section describes post-approval behavior only. No repository edits are made in the current phase.**

Once you have reviewed the proposal and verification checklist and are ready to apply the fix:

1. **Send an explicit instruction:** Type **APPROVE APPLY** (or equivalent) to authorize the agent or human to edit repository files.
2. **Apply the approved fix:** Implement the chosen fix option in the files identified in the proposal. Make only the edits described in the proposal (layout, classes, or structure). Do not significantly change visible layout intent; do not break keyboard navigation or a11y semantics.
3. **Re-verify in the environment:** In a browser at common desktop widths (e.g. 1024, 1280, 1440 px) and optionally on mobile:
   - Scroll from different areas of the page (content, sidebar regions); confirm main content scrolls as intended.
   - Confirm sticky sidebars and headers still behave correctly.
   - Confirm Tab order and focus behavior.
   - Confirm screen reader semantics are unchanged (e.g. test with accessibility tools if available).
4. **Optional:** Run any existing tests (e.g. docs or layout tests) and fix any regressions.
5. **Review as a pull request:** Ensure changes are explainable and reviewable; add a short PR description referencing this runbook and the chosen fix option. User may commit and push to their fork; do not open upstream PR.

Until you send **APPROVE APPLY**, the agent or human must **not** modify any repository files.

---

## Message flow (summary)

```
Invoker → DocsScrollContainerAnalyst (Analyze → Locate → Reproduce → Propose → Verify) → Invoker
```

---

## Artifact paths

- **Tool spec (machine-readable):** `tools/ashhq-docs-scroll-container-usability-fix.jsonld`
- **Runbook (this file):** `docs/ashhq-docs-scroll-container-usability-fix-runbook.md`

---

## References

- Target repo: ash_hq (Phoenix LiveView application).
- Target area: Documentation pages where main content scrolling is constrained.
- Goal: Improve scroll usability (full-page or widened scroll) while preserving layout intent, sticky sidebars, keyboard navigation, and accessibility. Apply behavior: PROPOSE ONLY until APPROVE APPLY.

Created using AALang and Gab
