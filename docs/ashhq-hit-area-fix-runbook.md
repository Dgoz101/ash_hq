# ashhq-hit-area-fix — Runbook

Developer workflow for improving UI accessibility (hit area) in Phoenix LiveView applications. This runbook can be followed by a human or an agent (e.g. Cursor Agent) to analyze a project, locate a small clickable area, apply a fix to reach ~44×44px without layout or visible changes, and verify the result.

**Product name:** ashhq-hit-area-fix  
**First use case:** ash_hq issue #34 — “Including packages” remove icon has a clickable area that is too small.

---

## Prerequisites

- Phoenix/LiveView project (e.g. ash_hq) at a known repo path (e.g. workspace root).
- Ability to edit project files (templates, components, CSS/utility classes) and to run/verify in a browser or test environment if needed.

---

## Constraints (must be respected)

- **No layout shifts** — Do not change spacing or layout of surrounding content.
- **No visible icon size change** — The icon’s visual size stays the same.
- **~44×44px hit area** — Clickable/touchable area should be at least approximately 44×44px (WCAG touch target).
- **Desktop and touch** — Solution must work for both pointer and touch.
- **Explainable and reviewable** — Changes should be clear and suitable for a pull request.

---

## Workflow (4 steps)

The pipeline has four logical steps, matching the AALang tool actors: **Analyzer → Locator → Fixer → Verifier**.

---

### Step 1 — Analyze (Analyzer)

**Goal:** Understand the Phoenix/LiveView project structure so the target UI element can be found.

1. Set the repo path (e.g. workspace root).
2. Identify:
   - LiveView/Web paths (e.g. `lib/*_web/`).
   - Component modules and where they live (e.g. `lib/*_web/components/`).
   - Template/layout locations (e.g. `.heex` under `lib/*_web/` or `templates/`).
   - Asset/CSS entry (e.g. `assets/`, Tailwind config).
3. Note the CSS/utility convention (e.g. Tailwind) and any project-specific patterns.
4. Produce a short **project structure summary** (paths and conventions) to pass to the next step.

**Output (conceptual):** `projectStructure` — repo path, liveViewPaths, componentPaths, assetPaths, cssFramework, conventions.

---

### Step 2 — Locate (Locator)

**Goal:** Find the exact UI element responsible for the small hit area (e.g. the “Including packages” remove icon for issue #34).

1. Use the project structure from Step 1.
2. Use the issue or task description (e.g. “Including packages remove icon”, “small clickable area”, “remove version button”) to search:
   - Component and page modules (e.g. search for “remove”, “version”, “packages”, “x-mark”, “close”).
   - Relevant `.heex` templates and any inline SVG or icon classes (e.g. `hero-x-mark`, `h-6 w-6`).
3. Identify:
   - **File path** (e.g. `lib/ash_hq_web/components/search.ex` or a template).
   - **Component or template name** and the **exact node** (button/link/icon wrapper).
   - **Current markup and CSS/utility classes** (e.g. `class="..."`).
   - A **stable selector** (e.g. id, data attribute, or component path) for verification.
4. Record a short **description** (e.g. “remove icon for Including packages”).

**Output (conceptual):** `elementLocation` — filePath, componentName, selector, currentMarkup, currentClasses, description.

---

### Step 3 — Fix (Fixer)

**Goal:** Propose and apply a fix that enlarges the clickable area to ~44×44px without changing visible layout or icon size.

1. Use the `elementLocation` from Step 2.
2. **Propose** a change that achieves ~44×44px hit area, for example:
   - Add padding (e.g. `p-2` or `p-3`) on the interactive element so the clickable area grows; keep the icon’s visual size (e.g. `h-6 w-6`) unchanged.
   - Or use `min-w-[44px] min-h-[44px]` (or equivalent) with `flex items-center justify-center` so the touch target is at least 44×44px.
   - Prefer utility classes already used in the project (e.g. Tailwind) and avoid custom CSS unless necessary.
3. **Apply** the fix by editing the repo files in place (e.g. the component or template from Step 2).
4. Record:
   - **Changed files** (paths).
   - **Patch summary** (what was changed in words).
   - **Rationale** (why this meets the constraints).
   - **Snippet** (the before/after or key lines) for the PR.

**Output (conceptual):** `fixApplied` — changedFiles, patchSummary, rationale, snippet.

---

### Step 4 — Verify (Verifier)

**Goal:** Confirm layout is unchanged and accessibility is improved (keyboard focus + screen reader).

1. Use the `fixApplied` (and optionally `elementLocation`) from Step 3.
2. **Layout:**
   - Confirm there is no layout shift (e.g. compare before/after or run the app and check the surrounding UI).
   - Confirm the visible icon size is unchanged.
3. **Hit area:**
   - Confirm the interactive element’s clickable/touch area is approximately 44×44px (e.g. via devtools or by measuring the element’s box).
4. **Keyboard:**
   - Confirm the element is focusable (e.g. `tabindex="0"` if needed, or native focusable element) and that focus is visible (e.g. focus ring).
5. **Screen reader:**
   - Confirm the control has an appropriate label (e.g. `aria-label="Remove"` or visible text) so screen reader users understand the action.
6. Fill a short **checklist** (criterion + passed yes/no) and note any **suggestions** (e.g. “add aria-label if missing”).

**Output (conceptual):** `verificationResult` — layoutUnchanged, keyboardFocusImproved, screenReaderImproved, checklist, suggestions.

---

## Message flow (summary)

```
Invoker → Analyzer → (projectStructure) → Locator → (elementLocation) → Fixer → (fixApplied) → Verifier → (verificationResult) → Invoker
```

---

## First run (ash_hq issue #34)

- **Target repo:** ash_hq (this project).
- **Target issue:** #34 — “Including packages” remove icon has a clickable area that is too small.
- **Steps:** Run Step 1 (analyze ash_hq), Step 2 (locate the remove icon, e.g. in search/version or package selector UI), Step 3 (apply padding or min-size to reach ~44×44px, no layout/icon size change), Step 4 (verify layout and a11y). Result should be reviewable as a single PR.

---

## References

- Tool spec (machine-readable): `tools/ashhq-hit-area-fix.jsonld`
- WCAG 2.2 Target Size (Level AAA): at least 44×44 CSS pixels for touch/pointer targets where possible.

Created using AALang and Gab
