# ashhq-package-version-indicator-overlap-fix — Runbook

Developer workflow for fixing a layout bug where the package version indicator overlaps headings/content on certain screen sizes and with long top headings in Phoenix LiveView documentation pages. This runbook can be followed by a human or an agent (e.g. Cursor Agent) to analyze the docs header layout, locate the version indicator markup and styling, reproduce the overlap, propose a fix (without editing repo files unless approved), and verify conceptually—**without editing repo files** unless the user explicitly approves.

**Product name:** ashhq-package-version-indicator-overlap-fix  
**Target repo:** ash_hq  
**Target area:** Documentation package header region where the version indicator is displayed  
**Immediate task:** Package version indicator is positioned in a way (often absolute) that can float on top of headings/content under certain desktop widths and when the top H1/title is long enough; propose a fix that prevents overlap while preserving design intent.

---

## Prerequisites

- Phoenix/LiveView project (ash_hq) at a known repo path (e.g. workspace root).
- Windows environment; Docker + local Phoenix server; Cursor Agent mode (or equivalent).
- Ability to read repo files (layouts, components, HEEx). No write access required unless approval is given (see Approval gate).

---

## Approval gate

**Do not edit repo files unless user types APPROVE APPLY.**

All steps in this runbook are **propose-only** until the user explicitly says "APPROVE APPLY." Until then, the agent or human must only analyze, locate, reproduce, propose (with concrete edits described in text/snippets), and verify conceptually. No file writes, no patches applied.

---

## Constraints (must be respected)

- **No visible layout intent change** — The on-screen appearance and design intent (e.g. indicator top-right) must be preserved after any future fix.
- **No break keyboard navigation** — Tab order and focus behavior must be preserved (or improved).
- **No break a11y semantics** — Screen reader landmarks, headings, and reading order must be preserved.
- **Propose only** — Do not modify repo code unless the user has typed APPROVE APPLY.
- **Minimal, localized changes** — Prefer changes limited to the header/indicator area; avoid global CSS unless unavoidable.
- **Explainable and reviewable** — The proposal and verification should be clear enough for a pull request.

---

## Workflow (5 steps)

The pipeline has five logical phases, executed by a single actor: **Analyze → Locate → Reproduce → Propose → Verify**.

---

### Step 1 — Analyze

**Goal:** Understand the page header layout where the package title/heading and version indicator are rendered (LiveView, HEEx, components, Tailwind).

1. Set the repo path (e.g. workspace root for ash_hq).
2. Identify the docs header area:
   - Routes and LiveViews that serve documentation (e.g. doc routes, doc layout).
   - Layout(s) or components that render the **package header** (title/H1 and version indicator).
   - Where the **version indicator** is rendered relative to the **title/heading**.
3. Note how they are composed: which layout or component includes the header row, and in what order (title vs indicator in the template).
4. Produce a short **layout summary**: which files/modules render the header, title, and version indicator, and how they are wired together.

**Output (conceptual):** `layoutSummary` — header, title, and version indicator render locations, components, composition.

---

### Step 2 — Locate

**Goal:** Locate the version indicator markup and styling: where it appears in the DOM and what positioning rules apply.

1. Use the layout summary from Step 1.
2. Find the version indicator in the markup:
   - **File path** and **component or template name** (e.g. docs page LiveView, layout, or shared component).
   - **DOM position** — which container holds the indicator and how it relates to the title/heading container.
   - **Positioning rules** — relative/absolute, z-index, overflow, padding, container constraints (e.g. flex/grid parent, or absolute within a relative wrapper).
3. Record a **stable selector** (e.g. class, data attribute, or structure) and the **current markup and Tailwind/CSS classes** for the indicator and its wrapper.
4. Record a clear **version indicator location description** (DOM position, positioning, z-index, overflow, padding, container).

**Output (conceptual):** `versionIndicatorLocation` — DOM position, positioning rules, z-index, overflow, padding, container constraints, filePath, selector, currentMarkup, currentClasses.

---

### Step 3 — Reproduce

**Goal:** Reproduce the overlap scenario (without committing fake content) and document clear local reproduction steps.

1. Use the version indicator location from Step 2.
2. **Identify breakpoint ranges** where overlap occurs (e.g. specific desktop widths in px or Tailwind breakpoints such as `md`/`lg`).
3. **Identify the content condition** that triggers overlap (e.g. long H1/title that wraps, pushing content under the absolutely positioned indicator).
4. **Document reproduction steps:**
   - **Route** — e.g. URL or doc page where the issue appears (e.g. a specific package doc with a long name).
   - **Browser width(s)** — e.g. 1024px, 1280px, or a range where overlap is visible.
   - **Content conditions** — e.g. “Use a package whose name is long enough to wrap” or “Resize to X px.”
5. Do **not** commit fake or placeholder content to the repo; use existing content or describe how to reproduce with real data.

**Output (conceptual):** `reproduction` — breakpoint ranges, content condition, reproduction steps (route, width, content).

---

### Step 4 — Propose

**Goal:** Propose (but do **not** apply yet) a fix that prevents overlap while preserving design intent.

1. Use the version indicator location and reproduction details from Steps 2 and 3.
2. **Propose** one or more fix options:
   - **Preferred:** Layout-driven solutions (e.g. flex or grid header row) so the version indicator participates in layout flow and no longer overlaps when the title wraps.
   - **Alternative:** Retain absolute positioning but reliably reserve space (e.g. padding/margin/containment on the title or container) without fragile magic numbers.
   - Ensure long headings wrap safely (e.g. `min-w-0`, `break-words`) and the indicator remains aligned and readable.
3. Document **tradeoffs** (e.g. layout change vs minimal CSS, maintainability, browser support).
4. Recommend a **preferred option** and document:
   - **Files to change** (e.g. layout HEEx, component, or Tailwind classes).
   - **Concrete edits** (snippet-level: what to add or change).
   - **Rationale** (why this meets constraints and prevents overlap).
5. Do **not** edit any repo files unless the user has typed APPROVE APPLY.

**Output (conceptual):** `proposal` — fix options with tradeoffs, preferred option, file-level changes, snippets, rationale.

---

### Step 5 — Verify

**Goal:** Verify conceptually (not by editing code yet) that the proposal would: eliminate overlap at common desktop widths, allow long headings to wrap without going under the indicator, preserve visual layout intent, preserve keyboard navigation, and preserve screen reader semantics.

1. Use the proposal from Step 4.
2. **No overlap:** Confirm that at common desktop widths (e.g. 1024, 1280, 1440 px) the indicator would not overlap the heading/content.
3. **Long headings wrap:** Confirm that long H1/title would wrap without going under the indicator (e.g. with `min-w-0`, `break-words`, or layout flow).
4. **Visual layout intent:** Confirm that the indicator would still appear top-right (or equivalent) and the overall design intent is preserved.
5. **Keyboard navigation:** Confirm that tab order and focus behavior would remain logical (e.g. focus order: title/heading, then indicator, or as appropriate).
6. **Screen reader semantics:** Confirm that landmarks, headings, and reading order would remain correct for assistive technologies.
7. Fill a short **verification checklist** (criterion + passed + notes) and any **caveats** or follow-up checks (e.g. “verify in browser after applying”).

**Output (conceptual):** `verificationChecklist` — noOverlap, longHeadingsWrap, layoutIntentPreserved, keyboardPreserved, a11yPreserved, plus notes.

---

## How to reproduce the issue

Use the outputs from **Step 3 (Reproduce)** to reproduce locally:

1. Start the app (e.g. Docker + local Phoenix server).
2. Open a docs page that shows the package header with version indicator (see **route** from reproduction).
3. Set browser width to a value where overlap occurs (see **breakpoint ranges** / **browser width** from reproduction).
4. If needed, use or navigate to content that triggers the condition (e.g. package with long name so the H1 wraps).
5. Observe: the version indicator overlaps the heading or content.

Record the exact route, width(s), and content condition so the fix can be validated against the same scenario.

---

## Diagnosis of root cause

After **Step 2 (Locate)** and **Step 3 (Reproduce)**, summarize the root cause:

- **Positioning:** e.g. version indicator uses absolute positioning and does not participate in layout flow, so when the title wraps or the container is narrow, content flows under the indicator.
- **Container/layout:** e.g. header is not a flex/grid row that reserves space for the indicator, or overflow/containment is missing.
- **Content:** e.g. long H1 with no `min-w-0` or `break-words` causes wrapping that overlaps the fixed-position indicator.

Document this in the runbook or in the analyst’s output so the proposal targets the correct cause.

---

## Proposed fix options with tradeoffs

Use the **Step 4 (Propose)** output. In the runbook or proposal, include:

- **Option A (e.g. layout-driven):** Flex or grid header row; title and indicator in flow; indicator no longer overlaps. *Tradeoffs:* may require small structural change; usually more robust.
- **Option B (e.g. reserved space):** Keep absolute positioning; add padding/margin or a spacer so content never goes under the indicator. *Tradeoffs:* minimal markup change; risk of magic numbers or breakpoints if not done carefully.
- **Recommendation:** State which option is preferred and why (e.g. layout-driven for long-term stability, or reserved space for minimal diff).
- **Snippets:** Concrete before/after or key class changes for the chosen option.

---

## Conceptual verification checklist

Before any code change, verify conceptually that the proposal satisfies:

| Criterion | Passed | Notes |
|-----------|--------|-------|
| No overlap at 1024, 1280, 1440 px | ☐ | |
| Long headings wrap without going under indicator | ☐ | |
| Visual layout intent preserved (indicator top-right or equivalent) | ☐ | |
| Keyboard navigation order logical | ☐ | |
| Screen reader semantics preserved | ☐ | |

Fill this table (or equivalent) as part of **Step 5 — Verify** and attach any caveats or follow-up checks (e.g. “confirm in browser after applying”).

---

## Future: applying the fix (after approval)

**This section describes post-approval behavior only. No repository edits are made in the current phase.**

Once you have reviewed the proposal and verification checklist and are ready to apply the fix:

1. **Send an explicit instruction:** Type **APPROVE APPLY** (or equivalent) to authorize the agent or human to edit repository files.
2. **Apply the approved fix:** Implement the chosen fix option (e.g. layout-driven or reserved-space) in the files identified in the proposal. Make only the edits described in the proposal (snippets, classes, structure).
3. **Re-verify in the environment:** In a browser at common desktop widths (e.g. 1024, 1280, 1440 px), with a long heading if applicable, confirm:
   - The version indicator no longer overlaps the heading/content.
   - Long headings wrap correctly and the indicator remains aligned and readable.
   - Visual layout intent is unchanged (indicator still top-right or equivalent).
   - Keyboard navigation order is logical.
   - Screen reader semantics are preserved (e.g. test with a screen reader or accessibility tools if available).
4. **Optional:** Run any existing tests (e.g. layout or docs tests) and fix any regressions.
5. **Review as a pull request:** Ensure changes are explainable and reviewable; add a short PR description referencing this runbook and the chosen fix option.

Until you send **APPROVE APPLY**, the agent or human must **not** modify any repository files.

---

## Message flow (summary)

```
Invoker → PackageVersionIndicatorOverlapAnalyst (Analyze → Locate → Reproduce → Propose → Verify) → Invoker
```

---

## Artifact paths

- **Tool spec (machine-readable):** `tools/ashhq-package-version-indicator-overlap-fix.jsonld`
- **Runbook (this file):** `docs/ashhq-package-version-indicator-overlap-fix-runbook.md`

---

## References

- Target repo: ash_hq (Phoenix LiveView application).
- Target area: Documentation package header region where the version indicator is displayed.
- Goal: Prevent package version indicator from overlapping headings/content at certain desktop widths and with long H1/title, while preserving design intent, keyboard navigation, and accessibility semantics.

Created using AALang and Gab
