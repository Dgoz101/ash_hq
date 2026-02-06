# ashhq-mobile-hamburger-reachability-fix — Runbook

Developer workflow for improving mobile usability and ergonomics in a Phoenix LiveView application. The **immediate task:** the hamburger (menu) icon is on the **left** side of the screen at mobile widths, which is harder to use for a large portion of users (right-handed, one-handed use with thumb in the lower-right area). This runbook can be followed by a human or an agent (e.g. Cursor Agent) to analyze the mobile header layout and ergonomics, reproduce the issue, propose one or more fixes (without editing repo files unless approved), and verify conceptually. **No repository edits** until the user explicitly types **APPROVE APPLY**.

**Product name:** ashhq-mobile-hamburger-reachability-fix  
**Target repo:** ash_hq  
**Target area:** Mobile header / navigation where the hamburger menu is rendered (docs page).  
**Immediate task:** Improve reachability of the docs hamburger menu for right-handed one-handed use; propose moving or duplicating the menu button to the right at mobile breakpoints while preserving design intent, keyboard navigation, and accessibility.

---

## Prerequisites

- Phoenix/LiveView project (ash_hq) at a known repo path (e.g. workspace root).
- Windows environment; Docker + local Phoenix server; Cursor Agent mode (or equivalent).
- Ability to read repo files (LiveView, HEEx, components, Tailwind). No write access required unless approval is given (see Approval gate).

---

## Approval gate

**Do not edit repo files unless user types APPROVE APPLY.**

All steps in this runbook are **propose-only** until the user explicitly says "APPROVE APPLY." Until then, the agent or human must only analyze, reproduce, propose (with concrete edits described in text/snippets), and verify conceptually. No file writes, no patches applied.

---

## Constraints (must be respected)

- **Do not significantly change visible layout intent outside of mobile** — Desktop and tablet layouts must remain unchanged; only mobile breakpoint behavior may change.
- **Do not break keyboard navigation** — Tab order and focus behavior for the menu button and header must be preserved.
- **Do not break a11y semantics** — Screen reader semantics and accessible names (e.g. for the menu button) must be preserved or improved.
- **Propose only** — Do not modify repo code unless the user has typed APPROVE APPLY.
- **Minimal, localized changes** — Prefer changes limited to the mobile header bar and hamburger placement; use mobile breakpoint–specific classes (e.g. `xl:` only where needed).
- **Explainable and reviewable** — The proposal and verification should be clear enough for a pull request. User may commit and push to their fork; do not open upstream PR.

---

## Workflow (5 steps)

The pipeline has five logical phases, executed by a single actor: **Analyze layout → Analyze ergonomics → Reproduce → Propose → Verify**.

---

### Step 1 — Analyze layout

**Goal:** Identify where the hamburger/menu button is rendered, how the mobile header is structured, and which breakpoints define “mobile” behavior.

1. Set the repo path (e.g. workspace root for ash_hq).
2. **Identify the mobile header and hamburger:**
   - **Routes/LiveViews** that render the docs page (e.g. `:docs_dsl` live action, `Docs` LiveComponent).
   - The **mobile header bar** that appears only below a certain width (look for classes like `xl:hidden`, `xl:block` on the sidebar vs mobile bar).
   - The **hamburger button** markup: which file, which component or template, and the element that toggles the sidebar (e.g. `phx-click={show_sidebar()}` or similar).
3. **Header structure:**
   - **Flex/grid:** Note the container classes (e.g. `flex flex-row justify-start` vs `justify-between`).
   - **Alignment:** Whether the menu button is the first child (left) or last (right), and what else is in the bar (logo, search, etc.).
   - **Breakpoints:** Which Tailwind breakpoint defines “mobile” (e.g. `xl` = 1280px default; below that the mobile bar is visible).
4. **Record:** File path(s), line or component name, selector or class for the mobile bar and for the hamburger button, and breakpoint value(s).

**Output (conceptual):** `layoutSummary` — hamburger render location (file, component, selector), mobile header structure (flex/grid, alignment), breakpoints (e.g. xl = 1280px).

**Current ash_hq finding (for reference):** The hamburger is in `lib/ash_hq_web/pages/docs.ex`. A docs-specific mobile header bar is the first child of the docs layout: `class="xl:hidden sticky top-20 z-40 h-14 ... flex flex-row justify-start w-full space-x-6 items-center ..."`. The menu button is the first element in that bar: `<button phx-click={show_sidebar()}>` with icon `<span class="hero-bars-3 w-8 h-8 ml-4" />`. “Mobile” is viewport &lt; **xl** (1280px). The global TopBar (`lib/ash_hq_web/components/app_view/top_bar.ex`) sits above this and is not the source of the hamburger; the hamburger is only on the docs page.

---

### Step 2 — Analyze ergonomics

**Goal:** Confirm menu position at mobile, identify other interactive elements, and document the one-handed usage scenario.

1. **Menu position:** Confirm the menu icon is on the **left** at mobile widths (first in flex order or left-aligned).
2. **Other header elements:** List any other interactive elements in the same mobile bar (e.g. logo link, search button, theme toggle). Note whether they are left, center, or right.
3. **Sticky/fixed:** Note if the header is `sticky` or `fixed` (e.g. `sticky top-20`). This keeps the header in view but does not change the fact that the left-side control is far from the right-hand thumb zone.
4. **One-handed scenario:** Document: “Most users are right-handed and often use phones one-handed, with the thumb in the **lower-right** area of the screen. The left-side menu button is at the **furthest ergonomic distance** from that thumb zone, making it harder to tap without shifting grip or using the other hand.”

**Output (conceptual):** `ergonomicsSummary` — menu position (left), other elements, sticky/fixed, one-handed scenario description.

**Current ash_hq finding:** The docs mobile bar is `sticky top-20 z-40`. The only visible control in that bar is the hamburger (left, `ml-4`). No other buttons or links in that row. The global TopBar above has logo (left), search icon and nav links (right) when on docs; the docs mobile bar is specifically for opening the docs sidebar.

---

### Step 3 — Reproduce

**Goal:** Document exact reproduction steps, one-handed scenario, and expected vs actual ease of access.

1. **URL/route:** e.g. any docs route where the mobile header is visible: `/docs/ash/latest/guide/...` or `/docs/ash/latest/...`.
2. **Viewport / device:** Set browser width &lt; 1280px (e.g. 375px, 390px, 414px for phone) or use DevTools device preset (e.g. iPhone, Pixel).
3. **Reproduction steps:**
   - Open the docs page at a mobile width.
   - Observe the sticky bar below the main top bar; the hamburger (three horizontal lines) is on the **left**.
   - Simulate one-handed use: hold phone in right hand, thumb in lower-right area; try to tap the hamburger without shifting grip — it is at the **furthest** reach.
4. **Expected vs actual:**
   - **Expected (better ergonomics):** Primary navigation control (menu) is in the **thumb-reachable** zone (e.g. right side or bottom) for right-handed one-handed use.
   - **Actual:** Menu is on the left; many users must stretch thumb, use left hand, or change grip to open the menu.

**Output (conceptual):** `reproduction` — URL/route, viewport width or device preset, exact steps, expected vs actual.

---

### Step 4 — Propose

**Goal:** Propose (but do **not** apply yet) one or more fixes that improve reachability while preserving design intent and accessibility.

1. **Preferred:** Move the hamburger/menu button to the **right** side at mobile breakpoints.
   - **Implementation idea:** Change the mobile header from `justify-start` to `justify-end`, or use `flex-row-reverse` so the button appears on the right while keeping DOM order for a11y if needed; or keep a left-aligned logo and add the button on the right (e.g. `justify-between` with logo left, button right).
   - **Tradeoffs:** Best for thumb reach; may affect visual balance (logo + empty space vs logo left / menu right). Ensure logo placement and branding remain intentional.
2. **Alternative:** Duplicate the menu button on the **right** for mobile only (keep left one for symmetry or a11y order; ensure only one is focusable or both open the same sidebar).
   - **Tradeoffs:** Improves reach without moving the left control; slightly more DOM and possible duplicate focus targets — need to handle focus and aria so only one “menu” is announced.
3. **Alternative:** Move primary navigation access to a **bottom-aligned** or thumb-reachable area (e.g. bottom nav bar) if consistent with design.
   - **Tradeoffs:** Larger design change; may not match current header-only pattern; good for full thumb zone.
4. **Constraints for any fix:**
   - **Visual hierarchy and branding:** Logo placement and header balance should remain intentional.
   - **Desktop/tablet:** No change at `xl` and above (sidebar visible, no mobile bar).
   - **Keyboard:** Focus order and Tab behavior must remain logical.
   - **Screen reader:** Accessible name for the menu button (e.g. aria-label) must be present and correct.
   - **Touch target size:** At least 44×44px clickable area and adequate spacing from adjacent controls.
5. Document **file-level changes** and **snippet-level** edits (before/after or key class/structure). Do **not** edit repo files unless user types APPROVE APPLY.

**Output (conceptual):** `proposal` — fix options with tradeoffs, preferred option, file-level changes, snippets, rationale.

**Current ash_hq proposal (preferred):** In `lib/ash_hq_web/pages/docs.ex`, for the mobile header bar (the `div` with `xl:hidden sticky top-20 ... flex flex-row justify-start ...`):
- **Option A (preferred):** Use `justify-between` and place the hamburger in a right-side wrapper. Structure: `[optional logo/placeholder left] [spacer] [button right]`. Or keep a single row with `justify-end` and the button as the only visible content on the right (logo can remain in the global TopBar above). Concretely: change `justify-start` to `justify-end`, and move the button to the end of the flex container (e.g. with `ml-auto` on the button or wrap the button in a div with `ml-auto`). Ensure the button has an accessible name (e.g. `aria-label="Open documentation menu"` or `aria-label="Open sidebar"`) — currently it may have no label.
- **Option B:** Duplicate the button: one left (current), one right `xl:hidden` with same `phx-click={show_sidebar()}`. Ensure one has `aria-hidden="true"` and is not in tab order, or use a single focusable control and the other decorative; avoid duplicate announcements.
- **Option C:** Document only; no code change (e.g. if design decision is to keep left for consistency with another product).

---

### Step 5 — Verify

**Goal:** Verify conceptually (no code edits yet) that the proposal satisfies reachability, no desktop/tablet regressions, keyboard, a11y, and touch target. Use the checklist below.

1. **Menu reachable with right-hand thumb:** After the fix, the primary menu control would be on the right (or in a thumb-reachable zone) so one-handed right-hand use can open the menu without stretching.
2. **No desktop/tablet regressions:** The mobile bar is `xl:hidden`; at 1280px and above the desktop sidebar is shown and the mobile bar is hidden. No change to TopBar or other layouts.
3. **Keyboard navigation:** Tab order: focus should reach the menu button in a logical order (e.g. after or before other header controls); Enter/Space should activate it. No duplicate focusable menu buttons unless one is `tabindex="-1"` and hidden from screen readers.
4. **Screen reader and accessible name:** The menu button must have an accessible name (e.g. aria-label). Verify that the proposal adds or preserves this (same requirement as in Step 4 — Propose).
5. **Touch target size and spacing:** Button or its clickable area should meet minimum touch target size (at least 44×44px) and adequate spacing from adjacent elements (same requirement as in Step 4 — Propose). Current icon is `w-8 h-8` (32px); padding or min-size may need to be ensured.
6. Fill the **conceptual verification checklist** (see below). Note any caveats or follow-up checks (e.g. "verify in browser after applying").

**Output (conceptual):** `verificationChecklist` — criteria plus passed/notes.

---

## How to reproduce the issue on mobile

Use the outputs from **Step 3 (Reproduce)**. Concrete steps for ash_hq:

1. Start the app (Docker + local Phoenix server).
2. Open a **docs** page, e.g. `/docs/ash/latest/guide/get-started` or any `/docs/...` route.
3. Set viewport width to **&lt; 1280px** (e.g. 375px, 390px) or use DevTools device toolbar (e.g. iPhone SE, Pixel 5).
4. Observe the **sticky bar** below the main top bar: it shows a **hamburger icon (three horizontal lines) on the left** with `ml-4`.
5. **One-handed use (right hand):** Hold the phone in your right hand with your thumb in the lower-right area. Try to tap the hamburger without changing grip — it is at the **furthest** ergonomic distance from the thumb.
6. **Expected (improved):** The menu control would be on the **right** (or in a thumb-reachable zone) so the same grip can open the menu easily.

Record the exact route and viewport used so the fix can be validated against the same scenario.

---

## Ergonomic rationale (one-handed use, thumb reach)

- **Right-handed majority:** A large portion of users are right-handed and hold the phone in the right hand.
- **Thumb zone:** In one-handed use, the thumb naturally rests and can comfortably reach the **lower-right** and **right edge** of the screen. The **top-left** is the **hardest** area to reach without shifting grip or using the other hand.
- **Current placement:** The hamburger is in the **top-left** (first item in a left-aligned flex row). That maximizes difficulty for right-handed one-handed use.
- **Design goal:** Place primary navigation controls (e.g. menu open) in the **thumb-reachable** zone — typically **right side** of the header at mobile — to improve usability without changing desktop layout.

---

## Proposed fix options with tradeoffs

| Option | Description | Tradeoffs |
|--------|-------------|-----------|
| **A (Preferred)** | Move the hamburger to the **right** at mobile breakpoints (e.g. `justify-end` or `justify-between` with button on right). | Best thumb reach; minimal DOM change; visual balance may shift (empty left or logo left / menu right). |
| **B** | **Duplicate** the menu button on the right for mobile; keep or hide the left one. | Improves reach; avoid duplicate focus/announcements (e.g. one `aria-hidden` + not in tab order). |
| **C** | **Bottom** or thumb-zone navigation (e.g. bottom nav bar). | Larger design change; may not match current header-only pattern. |

**Recommendation:** **Option A** is the preferred fix: move the hamburger to the right at mobile (single menu button on the right). Any chosen fix must include an accessible name (e.g. aria-label) for the menu button and a touch target size of at least 44×44px with adequate spacing, as in the Propose constraints and Verify checklist.

---

## Conceptual verification checklist

Before any code change, verify conceptually that the proposal satisfies these checks (use them in **Step 5 — Verify**):

| Criterion | Passed | Notes |
|-----------|--------|-------|
| Menu button is reachable with right-hand thumb in one-handed use (e.g. on the right at mobile). | ☐ | |
| No regressions to desktop or tablet layouts (mobile bar remains xl:hidden; desktop unchanged). | ☐ | |
| Keyboard navigation and focus order remain logical (Tab to menu button, Enter/Space activates). | ☐ | |
| Accessible name for the menu button intact or added (e.g. aria-label). | ☐ | |
| Touch target size at least 44×44px and adequate spacing. | ☐ | |

Fill this table (or equivalent) as part of **Step 5 — Verify**. All checks are conceptual only; no repo edits until APPROVE APPLY.

---

## Future: applying the fix (after approval)

**This section describes post-approval behavior only. No repository edits are made in the current phase.**

Once you have reviewed the proposal and verification checklist and are ready to apply the fix:

1. **Send an explicit instruction:** Type **APPROVE APPLY** (or equivalent) to authorize the agent or human to edit repository files.
2. **Apply the approved fix:** Implement the chosen fix option in the files identified in the proposal (e.g. `lib/ash_hq_web/pages/docs.ex`). Make only the edits described (mobile header layout, button placement, accessible name (e.g. aria-label), and touch target size/spacing per the Propose constraints). Do not change desktop/tablet layout; do not break keyboard or a11y.
3. **Re-verify in the environment:** In a browser at mobile width (e.g. 375px, 390px):
   - Confirm the menu button appears on the **right** (for Option A) or as chosen and is easy to tap with the right thumb.
   - Confirm at 1280px and above the mobile bar is hidden and the desktop sidebar is unchanged.
   - Confirm focus order and Tab behavior; confirm Enter/Space opens the sidebar.
   - Confirm the menu button has an accessible name (e.g. aria-label) and touch target size at least 44×44px.
4. **Optional:** Run any existing layout or a11y tests and fix regressions.
5. **Review as a pull request:** Ensure changes are explainable and reviewable; add a short PR description referencing this runbook and the chosen option. User may commit and push to their fork; do not open upstream PR.

Until you send **APPROVE APPLY**, the agent or human must **not** modify any repository files.

---

## Message flow (summary)

```
Invoker → MobileHamburgerReachabilityAnalyst (Analyze layout → Analyze ergonomics → Reproduce → Propose → Verify) → Invoker
```

---

## Artifact paths

- **Tool spec (machine-readable):** `tools/ashhq-mobile-hamburger-reachability-fix.jsonld`
- **Runbook (this file):** `docs/ashhq-mobile-hamburger-reachability-fix-runbook.md`

---

## References

- Target repo: ash_hq (Phoenix LiveView application).
- Target area: Mobile header / docs navigation where the hamburger menu is rendered.
- Goal: Improve reachability for right-handed one-handed use; preserve design intent, keyboard, and a11y. Apply behavior: PROPOSE ONLY until APPROVE APPLY.

Created using AALang and Gab
