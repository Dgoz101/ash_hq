# ashhq-docs-sidebar-dom-order-fix — Runbook

Developer workflow for improving in-page search usability on documentation pages in Phoenix LiveView applications. This runbook can be followed by a human or an agent (e.g. Cursor Agent) to analyze the docs layout, locate where the sidebar and main content appear in the DOM, propose a fix (DOM reorder + CSS grid/flex ordering) so main content comes first for CTRL/CMD+F, and verify conceptually—**without editing repo files** unless the user explicitly approves.

**Product name:** ashhq-docs-sidebar-dom-order-fix  
**Target repo:** ash_hq  
**Immediate task:** Docs sidebar is placed in markup before main content, so CTRL/CMD+F hits navigation matches before doc content; fix DOM order (visually unchanged) so content is found first.

---

## Prerequisites

- Phoenix/LiveView project (ash_hq) at a known repo path (e.g. workspace root).
- Windows environment; Docker + local Phoenix server; Cursor Agent mode (or equivalent).
- Ability to read repo files (layouts, components, HEEx). No write access required unless approval is given (see Approval gate).

---

## Approval gate

**Do not edit repo files unless user types APPROVE APPLY.**

All steps in this runbook are **propose-only** until the user explicitly says "APPROVE APPLY." Until then, the agent or human must only analyze, locate, propose (with concrete edits described in text/snippets), and verify conceptually. No file writes, no patches applied.

---

## Constraints (must be respected)

- **No visible layout change** — The on-screen appearance of the docs page must remain the same after any future fix.
- **No break keyboard navigation** — Tab order and focus behavior must be preserved (or improved).
- **No break a11y semantics** — Screen reader landmarks, headings, and reading order must be preserved.
- **Propose only** — Do not modify repo code unless the user has typed APPROVE APPLY.
- **Explainable and reviewable** — The proposal and verification should be clear enough for a pull request.

---

## Workflow (4 steps)

The pipeline has four logical phases, executed by a single actor: **Analyze → Locate → Propose → Verify**.

---

### Step 1 — Analyze

**Goal:** Understand the docs layout and how the sidebar and main content are rendered (LiveView, HEEx, components).

1. Set the repo path (e.g. workspace root for ash_hq).
2. Identify the docs area:
   - Routes and LiveViews that serve documentation (e.g. doc routes, doc layout).
   - Layout(s) that wrap docs pages (e.g. root layout, docs-specific layout).
   - Components that render the **sidebar** (nav, TOC, or doc index).
   - Components or templates that render the **main doc content**.
3. Note how they are composed: which layout includes which components, and in what order in the template (e.g. sidebar partial first, then main content).
4. Produce a short **layout summary**: which files/modules render sidebar vs main content, and how they are wired together.

**Output (conceptual):** `layoutSummary` — sidebar and main content render locations, components, composition.

---

### Step 2 — Locate

**Goal:** Locate where the sidebar markup is inserted relative to the main content in the DOM.

1. Use the layout summary from Step 1.
2. Trace the actual DOM order:
   - Which container (e.g. `<nav>`, `<aside>`, or wrapper div) holds the sidebar?
   - Which container holds the main content?
   - In the final HTML, does the sidebar container appear **before** or **after** the main content container?
3. Identify the file/layout/component and the exact template lines that produce this order (e.g. "sidebar partial is rendered first in layout X, then main content").
4. Record a clear **DOM order description** (e.g. "Sidebar is first in DOM, main content second; find-in-page hits sidebar links before doc body").

**Output (conceptual):** `domOrderDescription` — where sidebar vs main content appear in DOM, and which markup produces it.

---

### Step 3 — Propose

**Goal:** Propose a fix that keeps the visual layout the same while changing DOM order so main content comes before sidebar for find-in-page usability. **Do not apply the fix yet.**

1. Use the DOM order description from Step 2.
2. **Propose** a change that:
   - Renders or moves the **main content** block before the **sidebar** block in the DOM (e.g. reorder elements in the layout, or render main content first in the template).
   - Uses **CSS grid or flexbox ordering** (e.g. `order` or grid placement) so that on screen the sidebar still appears in the same visual position (e.g. left) and the main content in the same visual position (e.g. right or center).
   - Keeps landmarks and semantics intact (e.g. `<main>`, `<aside>`, or ARIA where appropriate).
3. Document the proposal:
   - **Files to change** (e.g. layout HEEx, or a wrapper component).
   - **Concrete edits** (snippet-level: what to reorder, what CSS to add or change).
   - **Rationale** (why this meets constraints and improves find-in-page).
4. Do **not** edit any repo files unless the user has typed APPROVE APPLY.

**Output (conceptual):** `proposal` — DOM reorder approach, CSS grid/flex ordering, file-level changes, snippets, rationale.

---

### Step 4 — Verify

**Goal:** Verify conceptually (not by editing code yet) that the proposal would: keep visual layout unchanged, preserve keyboard navigation, preserve screen reader semantics, and make CTRL/CMD+F hit main content first.

1. Use the proposal from Step 3.
2. **Visual layout:** Confirm that with the proposed DOM order and CSS ordering, the on-screen layout would remain the same (sidebar and content in the same positions).
3. **Keyboard navigation:** Confirm that tab order and focus behavior would be preserved or improved (e.g. focus still moves logically through sidebar and content).
4. **Screen reader semantics:** Confirm that landmarks, headings, and reading order would remain correct (e.g. main content still in `<main>`, sidebar in `<aside>` or equivalent).
5. **Find-in-page:** Confirm that with main content before sidebar in the DOM, CTRL/CMD+F would hit matches in the doc content before matches in the sidebar nav.
6. Fill a short **verification checklist** (criterion + passed + notes) and any **caveats** or follow-up checks (e.g. "verify in browser after applying").

**Output (conceptual):** `verificationChecklist` — layoutUnchanged, keyboardPreserved, a11yPreserved, findInPageContentFirst, plus notes.

---

## Message flow (summary)

```
Invoker → DocsSidebarDomOrderAnalyst (Analyze → Locate → Propose → Verify) → Invoker
```

---

## Artifact paths

- **Tool spec (machine-readable):** `tools/ashhq-docs-sidebar-dom-order-fix.jsonld`
- **Runbook (this file):** `docs/ashhq-docs-sidebar-dom-order-fix-runbook.md`

---

## References

- Target repo: ash_hq (Phoenix LiveView application).
- Goal: Improve in-page search (CTRL/CMD+F) by placing main doc content before sidebar in DOM, with visual layout unchanged via CSS ordering.

Created using AALang and Gab
