# ashhq-mobile-scroll-reset-on-navigation-fix — Runbook

Developer workflow for improving mobile navigation behavior on documentation pages in a Phoenix LiveView application. The **immediate task:** when navigating to a new documentation section (e.g. clicking a link in the docs sidebar or function list), the **scroll position does NOT reset to the top**; on mobile, users land partway down the page, which is disorienting and makes it harder to understand the new content. This runbook can be followed by a human or an agent (e.g. Cursor Agent) to analyze how docs section navigation works, identify where scroll state lives, reproduce the issue, propose one or more fixes (without editing repo files unless approved), and verify conceptually. **No repository edits** until the user explicitly types **APPROVE APPLY**.

**Product name:** ashhq-mobile-scroll-reset-on-navigation-fix  
**Target repo:** ash_hq  
**Target area:** Mobile docs navigation (sidebar links, function list, or in-page navigation).  
**Immediate task:** Ensure that when the user navigates to a new docs section on mobile, the view starts at the top of the new content; propose fix(es) that reset scroll on navigation without breaking desktop, back/forward, keyboard, or accessibility.

---

## Prerequisites

- Phoenix/LiveView project (ash_hq) at a known repo path (e.g. workspace root).
- Windows environment; Docker + local Phoenix server; Cursor Agent mode (or equivalent).
- Ability to read repo files (LiveView, HEEx, components, router, CSS). No write access required unless approval is given (see Approval gate).

---

## Approval gate

**Do not edit repo files unless user types APPROVE APPLY.**

All steps in this runbook are **propose-only** until the user explicitly says "APPROVE APPLY." Until then, the agent or human must only analyze, locate, reproduce, propose (with concrete edits described in text/snippets), and verify conceptually. No file writes, no patches applied. Future application of the fix is explicitly gated behind an "APPROVE APPLY" instruction.

---

## Constraints (must be respected)

- **Do not significantly change visible layout intent** — The on-screen appearance of the docs page (sidebars, headers, content placement) must remain the same after any future fix.
- **Do not break keyboard navigation** — Tab order and focus behavior must be preserved.
- **Do not break a11y semantics** — Screen reader semantics and reading order must be preserved.
- **Propose only** — Do not modify repo code unless the user has typed APPROVE APPLY.
- **Minimal, localized changes** — Prefer changes that only affect scroll reset on navigation (e.g. a hook, a single JS/LiveView callback, or a small layout tweak); avoid broad CSS or layout changes unless clearly justified.
- **Explainable and reviewable** — The proposal and verification should be clear enough for a pull request. User may commit and push to their fork; do not open upstream PR.

---

## Workflow (5 steps)

The pipeline has five logical phases: **Analyze navigation → Identify scroll container → Reproduce → Propose → Verify**.

---

### Step 1 — Analyze navigation

**Goal:** Understand how section navigation works on the docs page so you can later trigger a scroll reset at the right moment.

1. Set the repo path (e.g. workspace root for ash_hq).
2. **Identify how section navigation is triggered:**
   - **Sidebar links:** Where are docs sidebar links rendered? (e.g. `DocSidebar`, `lib/ash_hq_web/components/doc_sidebar.ex`.) Do they use `<.link href={...}>`, `live_patch`, `push_navigate`, or plain `<a href="...">`?
   - **Function list / in-page links:** If there is a right-hand function list or in-page anchors, how are those links implemented (patch, navigate, anchor hash)?
   - **Router:** Are docs under a `live` route (e.g. `live "/docs/...", AppViewLive, :docs_dsl`) or under a `get` route (e.g. `get "/docs/guides/:library/:version/*guide", HomeController, :home`)? This determines whether navigation is in-place (LiveView patch/navigate) or full page load.
3. **Determine content update model:**
   - If docs are under a **live** route: the same LiveView (e.g. `AppViewLive`) likely receives new params and re-renders the `Docs` LiveComponent; content swaps in place and **the same DOM/scroll container** is reused, so scroll position is preserved unless explicitly reset.
   - If docs are under a **get** route: each navigation is a full page load; scroll can still persist if the browser applies **scroll restoration** (e.g. `history.scrollRestoration = 'auto'`), so the new page may load with the previous scroll position.
4. Produce a **navigation summary**: trigger (patch/navigate/full load), routes involved, and whether the same LiveView/scroll container is reused.

**Relevant code (ash_hq):**  
- `DocSidebar` uses `<.link href={to} phx-click={mark_active(id)}>` where `to` comes from `DocRoutes.doc_link(guide)` (e.g. `/docs/guides/{library}/latest/{route}`).  
- Router: `get "/docs/guides/:library/:version/*guide", HomeController, :home` — docs guides are **GET** routes, so sidebar links typically cause a **full page load** unless the app uses a live route for docs in your fork.  
- If the app instead used a `live` route for docs, `<.link>` to that route would **patch** and the same LiveView would update; scroll would then be preserved because the document (or inner scroll container) is not reloaded.

**Output (conceptual):** `navigationSummary` — how section navigation is triggered (patch / push_navigate / full load), whether same LiveView or content swap, and which routes/LiveViews render docs.

---

### Step 2 — Identify the active scroll container (where scroll state lives)

**Goal:** Find the DOM element whose scroll position persists across navigation so you know what to scroll to top (e.g. `window`, `document.documentElement`, or a specific div).

1. Use the navigation summary from Step 1.
2. **On mobile (or narrow viewport), determine where scrolling happens:**
   - **Option A — DevTools:** Open a long docs page at mobile width. Right-click the main content → Inspect. Walk up the DOM from the content; the first ancestor that has `overflow: auto` or `overflow-y: auto` (or `overflow-y: scroll`) and a **constrained height** (e.g. `height`, `max-height`, or flex child with `min-h-0`) is the **scroll container**. If no such ancestor exists, the **window** (document/body) is the scroll context.
   - **Option B — Codebase:** Search for overflow and scroll-related classes on the docs layout: e.g. `overflow-y-auto`, `overflow-x-auto`, `overflow-y-hidden`, `sidebar-container`, `#docs-window`, `#main-container`. The docs content wrapper in `lib/ash_hq_web/pages/docs.ex` uses `#docs-window` with `overflow-x-auto overflow-y-hidden` — so the **main content column does not create vertical scroll**; vertical scroll is likely on the **window** or on a parent wrapper (e.g. `#main-container` or body). The `.sidebar-container` has `overflow-y-auto` and `max-height: calc(100vh - 8.5rem)` (see `assets/css/app.css`), so the **sidebar** is its own scroll context; the main content area may be in the document scroll context.
3. **Record:** Whether scroll is on **document/body** or on a **nested container**; id/class and file path of that element; and that this is the element whose scroll position persists when navigating to a new section (for in-place updates, the same container is reused; for full page load, the browser may restore scroll on the new page).

**Output (conceptual):** `scrollContainerLocation` — document vs nested container; DOM element (and selector); file paths and classes; note that this is where scroll state “lives” across navigation.

---

### Step 3 — Reproduce the issue

**Goal:** Reproduce the behavior so the fix can be validated: new section loads but scroll does not reset to top.

1. **Environment:** Mobile viewport (e.g. 375px width, or device toolbar in Chrome DevTools) or real device.
2. **Route:** Use a docs route that shows multiple sections or multiple guides (e.g. `/docs/guides/ash/latest/tutorials/get-started` or equivalent in your setup). Ensure you are on a page that uses the docs layout (e.g. Docs LiveComponent / docs_dsl if applicable).
3. **Steps:**
   - Open the docs page and scroll **down** so that the first screenful of the current section is no longer visible.
   - Click a link that navigates to a **different** section (e.g. another guide in the sidebar, or another item in the function list).
   - Observe: the new section content loads, but the **scroll position is unchanged** (or restored to the previous offset), so the user sees the middle/bottom of the new content instead of the top.
4. **Document:** Exact URL(s), viewport size, and “Expected: scroll resets to top of new section; Actual: scroll position preserved or restored to previous offset.”

**Output (conceptual):** `reproduction` — docs route(s), mobile viewport, exact steps, expected vs actual.

---

### Step 4 — Propose fix(es) (do not apply)

**Goal:** Propose one or more concrete fixes that reset scroll to top on section navigation, with tradeoffs. No repository edits.

**Recommended default behavior**

- **When scroll reset SHOULD happen:** On intentional forward navigation — e.g. the user taps a sidebar link or a function-list link to open a different docs section. In those cases, resetting to the top of the new content is the expected behavior.
- **When scroll reset SHOULD NOT happen:** On browser back/forward (history navigation) unless the product explicitly wants to reset there too. Users generally expect back/forward to restore the previous scroll position; forcibly resetting can feel broken.
- The preferred solution should distinguish between **user-initiated navigation** (sidebar/function list click → reset to top) and **history navigation** (back/forward → do not reset, or honor browser/restoration behavior) when that is feasible.

**Preferred: Reset scroll on LiveView navigation events**

- **When:** If docs are under a live route and navigation uses `patch` or `push_navigate`, reset scroll when the LiveView receives the new params (e.g. in `handle_params` or when the Docs LiveComponent updates).
- **How (conceptual):**
  - **Option A — Client hook:** A small `phx-hook` on the main content or layout that listens for LiveView navigation (e.g. `phx:navigate` or DOM updates that indicate new content) and sets `scrollTop = 0` on the active scroll container (window or the nested element identified in Step 2). If the scroll container is the window: `window.scrollTo(0, 0)` or `document.documentElement.scrollTop = 0`.
  - **Option B — LiveView JS:** After updating the docs content (e.g. in the LiveView that renders the Docs component), push a small JS command that scrolls the scroll container to top (e.g. `JS.dispatch("scroll-to-top")` to a hook that performs the scroll).
  - **Option C — Phoenix LiveView built-in:** If using a recent Phoenix LiveView version that supports scroll restoration or a dedicated callback for “scroll to top on patch,” use that when the patch is for a new docs section.
- **Tradeoffs:** Keeps navigation logic in one place; works for in-place updates; may need to avoid resetting when the user used browser back/forward (if you want to preserve scroll on back/forward, only reset on “new” navigations, e.g. sidebar click, not on `popstate`).

**Alternative: Reset when main content container changes**

- If the main content is re-rendered (e.g. new `id` or a wrapper that mounts/unmounts), a hook on that container could run on `updated()` and set scroll to top for the appropriate scroll container. Same tradeoffs as above regarding back/forward.

**Alternative: Single scroll context (document/body)**

- If the current design uses a **nested** scroll container for the main content and that causes both “narrow scroll area” and “scroll not reset,” consider whether moving scroll to the **document/body** (single scroll context) is acceptable. Then the whole page scrolls as one; a full page load would start at top by default, and for live patch you still need an explicit scroll-to-top on navigation. This may be a larger layout change; only propose if it aligns with project goals and the existing scroll-container runbook (ashhq-docs-scroll-container-usability-fix).

**Requirements for all options:**

- **Mobile:** The fix must work at mobile viewport (and optionally at all viewports).
- **Desktop:** Must not negatively affect desktop (e.g. no double-scroll or broken layout).
- **Back/forward:** Decide whether back/forward should restore scroll (often yes) or always start at top; document the choice in the proposal.

**Output (conceptual):** `proposal` — one or more fix options with short description, where to change (file/module/hook name), and tradeoffs; preferred option clearly marked; **no edits applied** until APPROVE APPLY.

---

### Step 5 — Verify conceptually (no code edits)

**Goal:** Check that the proposed fix(es) would satisfy requirements and not introduce regressions.

Use this checklist (conceptual; no code edits yet):

| Criterion | Passed | Notes |
|----------|--------|--------|
| On mobile, navigating to a new section always starts at the top of the content | ☐ | After applying the proposed fix, user lands at top of new section when using sidebar or function list. |
| No regressions to back/forward navigation | ☐ | Browser back/forward either restores previous scroll (if desired) or at least does not break. |
| Keyboard navigation and accessibility semantics remain intact | ☐ | Tab order, focus, and screen reader flow unchanged; scroll reset does not trap focus or skip content. |
| No layout shifts or visual jank | ☐ | Resetting scroll does not cause visible jump on desktop or mobile beyond the intended “start at top.” |

**Output (conceptual):** `verificationChecklist` — filled table and any caveats (e.g. “back/forward will always start at top if we reset on every patch”).

---

## Future: Applying the fix (after approval)

**Do not perform this section until the user explicitly types "APPROVE APPLY."**

1. Re-read the **proposal** from Step 4 and choose the agreed option.
2. Implement the change in the repo (e.g. add or update a hook, add a JS command in the LiveView, or adjust layout/CSS if the chosen option requires it).
3. Test locally: reproduce the scenario from Step 3 and confirm scroll resets to top on mobile when navigating to a new section.
4. Run through the verification checklist from Step 5 and fix any regressions.
5. Optionally document the change in a short PR description or commit message for the user to push to their fork.

---

## Summary table (example — fill from analysis)

| Item | Finding |
|------|--------|
| **Navigation trigger** | e.g. Full page load (GET) / LiveView patch / push_navigate |
| **Scroll container** | e.g. Window (document) / or nested div (selector and file) |
| **Reproduce** | e.g. /docs/guides/ash/latest/…, 375px width, scroll down → click other guide → scroll unchanged |
| **Preferred fix** | e.g. Hook that scrolls window to top on LiveView patch for docs |
| **Verification** | Checklist above; note any caveats |

---

**Product name:** ashhq-mobile-scroll-reset-on-navigation-fix  
**Target repo:** ash_hq  
**Target area:** Mobile docs navigation (sidebar, function list, in-page).  
**Apply behavior:** PROPOSE ONLY until user types **APPROVE APPLY**.
