# ashhq-mobile-hamburger-state-indicator-fix — Runbook

Developer workflow for improving mobile navigation usability on documentation pages in a Phoenix LiveView application. The **immediate task:** the hamburger (menu) icon does **not** show state — when the docs sidebar is open, the icon does not update to reflect an active/open state, making system state unclear. This runbook can be followed by a human or an agent (e.g. Cursor Agent) to analyze mobile docs navigation behavior, locate how open state is represented, reproduce the missing-state issue, propose one or more fixes (without editing repo files unless approved), and verify conceptually. **No repository edits** until the user explicitly types **APPROVE APPLY**.

**Product name:** ashhq-mobile-hamburger-state-indicator-fix  
**Target repo:** ash_hq  
**Target area:** Docs mobile header / sidebar toggle button.  
**Immediate task:** Add clear state feedback for the hamburger: when the sidebar is open, the button should indicate “open” (e.g. icon change to X, or active styling). Optional: improve ARIA (aria-expanded, aria-controls, aria-label). Preserve design intent, keyboard navigation, and accessibility.

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
- **Do not break keyboard navigation** — Tab order, Enter/Space toggle, and focus behavior for the menu button and header must be preserved.
- **Do not break a11y semantics** — Screen reader semantics and accessible names must be preserved or improved (e.g. aria-expanded, aria-label).
- **Propose only** — Do not modify repo code unless the user has typed APPROVE APPLY.
- **Minimal, localized changes** — Prefer changes limited to the docs mobile header and sidebar toggle (button icon/styling and optional ARIA).
- **Explainable and reviewable** — The proposal and verification should be clear enough for a pull request. User may commit and push to their fork; do not open upstream PR.

---

## Workflow (5 steps)

The pipeline has five logical phases, executed by a single actor: **Analyze navigation behavior → Locate state → Reproduce → Propose → Verify**.

---

### Step 1 — Analyze mobile docs navigation behavior

**Goal:** Identify where the hamburger/menu button is rendered, what it toggles, and how state is tracked.

1. Set the repo path (e.g. workspace root for ash_hq).
2. **Identify the hamburger/menu button:**
   - **Routes/LiveViews** that render the docs page (e.g. docs LiveComponent, docs layout).
   - The **mobile header bar** that appears below a certain width (e.g. `xl:hidden`).
   - The **button** that toggles the sidebar: file path, component or template, and the element (e.g. `phx-click={show_sidebar()}` or similar).
3. **What it toggles:** The sidebar (or overlay) that opens and closes — e.g. `#mobile-sidebar-container` or equivalent.
4. **How state is tracked:** Determine whether open/closed is tracked by: LiveView assign (e.g. `@sidebar_open?`), conditional rendering, CSS class (e.g. `hidden` on the sidebar container), or JS/LiveView JS (e.g. `JS.toggle`, `JS.show`/`JS.hide`). Note: the **button** may not receive any state-dependent class or assign; that is the issue.
5. **Record:** File path(s), line or component name, selector or class for the mobile bar and hamburger button; event names (e.g. `show_sidebar`, `hide_sidebar`); and how sidebar visibility is toggled (assign vs CSS vs JS).

**Output (conceptual):** `layoutSummary` — hamburger render location (file, component, selector), what it toggles (sidebar container id/class), how state is tracked (assign / CSS / JS).

**Current ash_hq finding (for reference):** The hamburger is in `lib/ash_hq_web/pages/docs.ex`. The mobile header bar is the first child of the docs layout: `class="xl:hidden sticky top-20 z-40 h-14 ... flex flex-row justify-end ..."`. The menu button is a `<button type="button" aria-label="Open documentation menu" class="min-h-11 min-w-11 ..." phx-click={show_sidebar()}>` with icon `<span class="hero-bars-3 w-8 h-8" />`. A separate hidden button `id={"#{@id}-hide"}` has `phx-click={hide_sidebar()}`. The sidebar is `#mobile-sidebar-container` with initial `class="hidden fixed ..."`. `show_sidebar/1` uses `JS.toggle` on `#mobile-sidebar-container` (opacity transition); `hide_sidebar/1` uses `JS.hide` on the same element. **State is not in an assign** — it is effectively “sidebar visible if container does not have `hidden`”. The **hamburger button never changes**: it always shows `hero-bars-3` and the same `aria-label="Open documentation menu"`, so there is no visual or ARIA indication of open state.

---

### Step 2 — Locate how “open” state is represented today

**Goal:** Determine exactly how the sidebar’s open/closed state is represented and why the button does not reflect it.

1. **Sidebar visibility:** Is the sidebar shown/hidden via: assigns (e.g. `@show_sidebar`), conditional rendering (`:if`), CSS class toggle (`hidden`), or LiveView JS (`JS.toggle` / `JS.show` / `JS.hide`)?
2. **State variables:** Are there any assigns or state variables that represent “sidebar open”? (e.g. `sidebar_open?`, `mobile_menu_open`). If the sidebar is driven only by JS (DOM class), there may be **no** server-side assign — the button cannot easily “know” open state unless it is derived (e.g. from a hook or from toggling the same element).
3. **Events and hooks:** List all relevant events: `phx-click` on the menu button, `phx-click-away` on the sidebar, any `phx-hook` that might track or sync state. Note any `id` used for the sidebar container (needed for `aria-controls`).
4. **Button behavior:** Does the button receive any class or attribute that changes when the sidebar opens? (e.g. `aria-expanded`, a class like `is-open`). In the current implementation, typically it does **not** — hence the missing state indicator.

**Output (conceptual):** `stateLocation` — how sidebar visibility is implemented (assigns / CSS / JS), any state variables, event names, hook names, and whether the button has any state-dependent attributes.

**Current ash_hq finding:** Sidebar visibility is controlled by **LiveView JS only**: `show_sidebar` uses `JS.toggle(to: "#mobile-sidebar-container", in: {...}, out: {...})` and `hide_sidebar` uses `JS.hide(to: "#mobile-sidebar-container", transition: {...})`. The container starts with `class="hidden ..."` and is toggled by these JS commands. There is **no** LiveView assign for “sidebar open”. The button has a fixed `aria-label="Open documentation menu"` and always shows `hero-bars-3`; no `aria-expanded`, no `aria-controls`, and no class change when the sidebar is open. The sidebar has `phx-click-away={hide_sidebar()}`. So: state lives in the **DOM** (presence/absence of `hidden` on `#mobile-sidebar-container`), and the button is not wired to that state.

---

### Step 3 — Reproduce the issue on mobile

**Goal:** Document exact steps to reproduce the missing-state issue and record expected vs actual.

1. **URL/route:** Any docs route where the mobile header is visible, e.g. `/docs/ash/latest/guide/get-started` or any `/docs/...` route.
2. **Viewport / device:** Set browser width &lt; 1280px (e.g. 375px, 390px, 414px) or use DevTools device preset (e.g. iPhone SE, Pixel 5).
3. **Reproduction steps:**
   - Open the docs page at a mobile width.
   - Observe the sticky bar: the **hamburger icon (three horizontal lines)** is visible (e.g. on the right in current layout).
   - **Tap the hamburger** to open the docs sidebar.
   - **Observe the icon:** It does **not** change — no X, no “pressed” or “active” styling. The sidebar is open but the button still looks like “closed”.
   - Close the sidebar (tap outside or use the hidden hide button if exposed). The icon still looks the same.
4. **Expected vs actual:**
   - **Expected:** When the sidebar is **open**, the button clearly indicates “open” (e.g. icon changes to X, or button has active/pressed styling). When **closed**, it shows the hamburger. This makes system state clear.
   - **Actual:** The icon never changes; users cannot tell from the button alone whether the sidebar is open or closed.

**Output (conceptual):** `reproduction` — docs route, mobile width or device preset, exact steps, expected vs actual.

---

### Step 4 — Propose (do not apply yet)

**Goal:** Propose one or more fixes to provide clear state feedback while preserving design intent and accessibility.

1. **Preferred: Icon swap**
   - When the sidebar is **closed**: show the **hamburger** icon (e.g. `hero-bars-3`).
   - When the sidebar is **open**: show a **close “X”** icon (e.g. `hero-x-mark` or equivalent).
   - **Implementation note:** This requires the button (or the LiveView) to “know” open state. Options: (a) Add an assign (e.g. `@mobile_sidebar_open`) updated in the same JS callback or via a small hook that syncs from the DOM; (b) Use a single button that toggles and swaps icon based on that assign; or (c) Use two buttons (one “open”, one “close”) and show/hide them based on state — ensure only one is focusable and only one is announced.
   - **Tradeoffs:** Very clear state; may require adding an assign and ensuring it stays in sync with JS toggle (e.g. via `phx-hook` or by replacing pure JS toggle with LiveView-driven show/hide that updates the assign).

2. **Alternative: Active visual state**
   - Keep the **hamburger** icon for both states.
   - When the sidebar is **open**, add an “active” visual state: e.g. different background, border, or color (e.g. `bg-base-light-200 dark:bg-base-dark-750` or a distinct “pressed” style). Ensure contrast meets accessibility requirements.
   - **Tradeoffs:** No icon asset change; state is communicated by styling only — ensure it is obvious and passes contrast.

3. **Optional: ARIA**
   - **aria-expanded:** Set `aria-expanded="true"` when sidebar is open, `aria-expanded="false"` when closed. Requires the element that has `aria-expanded` to reflect current state (assign or hook).
   - **aria-controls:** Set `aria-controls="mobile-sidebar-container"` (or the actual id of the sidebar container) on the button.
   - **aria-label:** Use dynamic label: e.g. “Open documentation menu” when closed and “Close documentation menu” when open, so assistive tech users hear the current action.
   - **Tradeoffs:** Best for screen reader users; must be kept in sync with actual open/closed state.

4. **Requirements for any fix:**
   - **Focus styles:** Keep focus ring/outline visible so keyboard users can see focus.
   - **Touch target:** Maintain at least ~44×44px clickable area and adequate spacing.
   - **Desktop/tablet:** No change at `xl` and above (mobile bar and toggle are hidden).
   - **Minimal scope:** Changes only in the docs mobile header and sidebar toggle area.

5. Document **file-level changes** and **snippet-level** edits (before/after or key attributes). Do **not** edit repo files unless user types APPROVE APPLY.

**Output (conceptual):** `proposal` — fix options with tradeoffs, preferred option, ARIA notes, file-level/snippet-level changes, rationale.

#### Recommended implementation approach

Given current ash_hq state is **DOM-driven** (JS.toggle/JS.hide on `#mobile-sidebar-container`, no LiveView assigns), the **lowest-risk** approach is a **client-side** solution that keeps state in the DOM and avoids adding assigns or server round-trips:

- Use a **small client-side mechanism** (e.g. a `phx-hook` or script) that observes or reacts to the sidebar container’s visibility (e.g. when the existing `show_sidebar`/`hide_sidebar` JS runs). When the container becomes visible: update the **single** menu button’s visual state (e.g. toggle a `data-state="open"` or class, swap icon from hamburger to X) and set `aria-expanded="true"`, `aria-label="Close documentation menu"`. When the container becomes hidden: set `data-state="closed"` (or remove the class), show hamburger icon, `aria-expanded="false"`, `aria-label="Open documentation menu"`. No second button, no duplicate focus targets.
- **Success criteria for this approach:**
  - **One button only** — single focusable toggle; no duplicate “open”/“close” buttons in the tab order.
  - **aria-controls** points to `#mobile-sidebar-container` (the id of the sidebar container).
  - **aria-label** switches between “Open documentation menu” (closed) and “Close documentation menu” (open).
  - **aria-expanded** is `true` when the sidebar is visible, `false` when hidden.
- **Tradeoff:** If we later need server-side state (e.g. for analytics or other features), we can refactor to an assign and event-driven show/hide. The initial fix should avoid that complexity and stay client-side so it works with the existing JS.toggle/JS.hide behavior without changing LiveView state.

This remains **propose-only** and **approval-gated**; no repo edits until APPROVE APPLY.

---

### Step 5 — Verify (conceptual only)

**Goal:** Verify conceptually (no code edits yet) that the proposal would satisfy visual state, a11y, keyboard, and layout. Use the checklist below.

1. **Visual state:** When the sidebar is open, the button clearly indicates “open” (icon or active styling). When closed, it indicates “closed”.
2. **Assistive tech:** If ARIA is proposed, `aria-expanded` reflects state and the label (e.g. aria-label) reflects the action (Open vs Close). Screen reader users can perceive state.
3. **Desktop/tablet:** No regressions; mobile bar remains `xl:hidden`; desktop layout unchanged.
4. **Keyboard:** Tab reaches the menu button; Enter/Space toggles the sidebar; focus remains sensible (e.g. no focus trap issues); if Escape closes the sidebar, that behavior still works.
5. **Layout stability:** Mobile layout does not shift or jump when toggling (e.g. no unexpected reflow from icon swap or class changes).
6. Fill the **conceptual verification checklist** (see below). Note any caveats or follow-up checks (e.g. “verify in browser after applying”).

**Output (conceptual):** `verificationChecklist` — criteria plus passed/notes.

---

## How to reproduce the missing-state issue on mobile

Use the outputs from **Step 3 (Reproduce)**. Concrete steps for ash_hq:

1. Start the app (Docker + local Phoenix server).
2. Open a **docs** page, e.g. `/docs/ash/latest/guide/get-started` or any `/docs/...` route.
3. Set viewport width to **&lt; 1280px** (e.g. 375px, 390px) or use DevTools device toolbar (e.g. iPhone SE, Pixel 5).
4. Observe the **sticky bar** below the main top bar: it shows the **hamburger icon (three horizontal lines)**.
5. **Tap the hamburger** to open the docs sidebar. The sidebar slides in (or fades in).
6. **Look at the hamburger button:** The icon is **unchanged** — still three lines. There is no X, no “pressed” or “active” styling. The sidebar is open but the button does not reflect that.
7. Close the sidebar (tap outside the sidebar or use the close mechanism). The button still looks the same.
8. **Expected:** When open, the button should show an X or an active state so users know the menu is open. When closed, show the hamburger. **Actual:** No visual change in either state.

Record the exact route and viewport used so the fix can be validated against the same scenario.

---

## Where state lives (assigns / events / hooks)

Summary for ash_hq (from Steps 1–2):

| Aspect | Current implementation |
|--------|-------------------------|
| **Sidebar visibility** | LiveView JS: `JS.toggle` / `JS.hide` on `#mobile-sidebar-container`. Container has class `hidden` when closed; JS removes/adds it. |
| **Server-side state** | **None** for “sidebar open”. No assign like `@mobile_sidebar_open` or `@sidebar_open?`. |
| **Button events** | `phx-click={show_sidebar()}` on the visible button; `phx-click={hide_sidebar()}` on a hidden button `#{@id}-hide`. Sidebar has `phx-click-away={hide_sidebar()}`. |
| **Button appearance** | Single icon `hero-bars-3` always; no class or attribute changes when sidebar opens. |
| **ARIA** | `aria-label="Open documentation menu"` only; no `aria-expanded`, no `aria-controls`. |
| **Hooks** | No phx-hook on the toggle or sidebar that syncs state back to the server. |

To add a state indicator, we need either: (a) an assign that tracks open/closed and is updated when we show/hide the sidebar (e.g. by moving show/hide into LiveView events that set the assign and optionally still use JS for transition), or (b) a client-side hook that toggles the button’s icon/ARIA based on the sidebar container’s visibility (e.g. MutationObserver or listening to the same JS transitions).

---

## Proposed fix options with tradeoffs

| Option | Description | Tradeoffs |
|--------|-------------|-----------|
| **A (Preferred)** | **Icon swap:** Hamburger when closed, close “X” when open. Requires knowing open state (assign or hook). | Clearest state; may need assign + event-driven show/hide or a small hook to sync DOM state to button. |
| **B** | **Active styling:** Keep hamburger icon; when open, add “active” style (background/border/color). Ensure contrast. | No icon asset change; state by style only — must be obvious and meet contrast. |
| **C** | **ARIA only:** Add `aria-expanded`, `aria-controls`, and dynamic `aria-label` (Open/Close). | Best for assistive tech; still no visual change unless combined with A or B. |

**Recommendation:** Prefer **Option A** (icon swap) for clear visual state; add **Option C** (ARIA) for accessibility. Option B is a valid alternative if icon swap is deferred. Any fix must preserve focus styles and ~44×44px touch target and remain localized to the docs mobile header and sidebar toggle.

---

## Conceptual verification checklist

Before any code change, verify conceptually that the proposal satisfies these checks (use them in **Step 5 — Verify**):

| Criterion | Passed | Notes |
|-----------|--------|-------|
| When sidebar is open, the button clearly indicates “open” state visually (icon or active styling). | ☐ | |
| Assistive tech can perceive state (e.g. aria-expanded and label reflects Open vs Close). | ☐ | |
| No regressions to desktop or tablet layouts (mobile bar remains xl:hidden; desktop unchanged). | ☐ | |
| Keyboard: Tab → menu button, Enter/Space toggles; focus remains sensible; Escape (if present) still works. | ☐ | |
| Mobile layout remains stable (no shifting/jumping when toggling). | ☐ | |
| Focus styles visible; touch target ~44×44px. | ☐ | |

Fill this table (or equivalent) as part of **Step 5 — Verify**. All checks are conceptual only; no repo edits until APPROVE APPLY.

---

## Future: applying the fix (after approval)

**This section describes post-approval behavior only. No repository edits are made in the current phase.**

Once you have reviewed the proposal and verification checklist and are ready to apply the fix:

1. **Send an explicit instruction:** Type **APPROVE APPLY** (or equivalent) to authorize the agent or human to edit repository files.
2. **Apply the approved fix:** Implement the chosen fix in the files identified in the proposal (e.g. `lib/ash_hq_web/pages/docs.ex`). Make only the edits described: state indicator (icon swap and/or active styling), optional ARIA (`aria-expanded`, `aria-controls`, dynamic `aria-label`), and ensure focus styles and touch target (~44×44px) are preserved. Do not change desktop/tablet layout; do not break keyboard or a11y.
3. **Re-verify in the environment:** In a browser at mobile width (e.g. 375px, 390px):
   - Open and close the sidebar; confirm the button clearly shows “open” when the sidebar is open and “closed” when it is closed.
   - Confirm at 1280px and above the mobile bar is hidden and the desktop layout is unchanged.
   - Confirm Tab → menu button, Enter/Space toggles, focus and (if present) Escape behavior.
   - Confirm ARIA state and label if implemented; confirm touch target and focus visibility.
4. **Optional:** Run any existing layout or a11y tests and fix regressions.
5. **Review as a pull request:** Ensure changes are explainable and reviewable; add a short PR description referencing this runbook and the chosen option. User may commit and push to their fork; do not open upstream PR.

Until you send **APPROVE APPLY**, the agent or human must **not** modify any repository files.

---

## Message flow (summary)

```
Invoker → HamburgerStateIndicatorAnalyst (Analyze → Locate state → Reproduce → Propose → Verify) → Invoker
```

---

## Artifact paths

- **Tool spec (machine-readable):** `tools/ashhq-mobile-hamburger-state-indicator-fix.jsonld`
- **Runbook (this file):** `docs/ashhq-mobile-hamburger-state-indicator-fix-runbook.md`

---

## References

- Target repo: ash_hq (Phoenix LiveView application).
- Target area: Docs mobile header / sidebar toggle button.
- Goal: Add clear state feedback for the hamburger (icon or active styling + optional ARIA); preserve design intent, keyboard, and a11y. Apply behavior: PROPOSE ONLY until APPROVE APPLY.

Created using AALang and Gab
