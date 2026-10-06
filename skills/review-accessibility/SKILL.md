---
name: review-accessibility
description: Web accessibility (WCAG 2.2 AA) at both ends of a change. Requirements mode turns a UI story or feature into testable a11y acceptance criteria before any code is written; review mode checks the staged diff's components, templates, and styles for a11y defects, reports them grouped by impact, and fixes them. Delegates WCAG guidance to the third-party `accessibility` skill. Use when writing or researching a UI story, when reviewing frontend changes, or when asked about accessibility, a11y, keyboard or screen-reader support, focus management, contrast, or WCAG. Web only — not React Native.
metadata:
  author: Bruno Zaninello
  version: "1.0.0"
---

# Review Accessibility

Accessibility for web UI, in two modes:

- **Requirements** — before implementation. A UI story or feature in, testable a11y acceptance
  criteria out. This is where most of the value is: a dialog designed to return focus costs nothing;
  one retrofitted after review costs a second pass through every gate.
- **Review** — after implementation. The staged diff in, findings grouped by impact out, then fixes.

Pick the mode from what you were handed. A story, PRD, or feature description with no code yet →
requirements. A diff, or a request to review/audit → review. If the invoking skill named a mode, use
that.

**Scope: web only.** HTML, React/JSX, Vue, Svelte, server-rendered templates. React Native and
native mobile are out of scope — if the change is RN, say so and stop.

## Knowledge source

Before either mode, invoke the `accessibility` skill by name via the Skill tool. It is the authority
for WCAG 2.2 success criteria, the POUR guidance, code patterns, and the evidence-led audit workflow
(Lighthouse, accessibility-tree snapshot, keyboard pass). Cite the success criterion behind each
criterion or finding — `WCAG 2.4.3 Focus Order`, not "best practice".

It is a third-party skill installed separately — see the Dependencies section of the
my-agentic-workflow README. If it isn't installed, proceed against WCAG 2.2 AA from your own
knowledge and say at the top of your output that the reference skill was missing.

**Target: WCAG 2.2 Level AA**, unless the repo's `CLAUDE.md` / `AGENTS.md` sets a different one.
AAA criteria are never requirements and never findings above Low.

## Find the repo's accessible primitives first

Both modes start here. Most a11y defects in a mature codebase are a hand-rolled version of something
the repo already does correctly. Look for, with `file:line`:

- A component library or headless primitives (Radix, React Aria, Headless UI, shadcn/ui, Reach)
- The project's own Dialog/Modal, Drawer, Popover, Menu, Tabs, Combobox
- A form-field wrapper that wires `<label>`, error text, and `aria-describedby`
- A toast or announcer utility (an `aria-live` region), a `VisuallyHidden` / `sr-only` helper
- Focus utilities: focus trap, focus-return, skip link, route-change focus handling
- Lint and test tooling already present: `eslint-plugin-jsx-a11y`, `@axe-core/playwright`,
  `jest-axe`, Testing Library

Requirements name these as the way to meet a criterion. Review flags bypassing them.

## Requirements mode

Input: one UI story (or a feature's user flows). Output: the content of the story's
**Accessibility** section — criteria and test scenarios — in the shape the `write-user-stories`
UI template expects.

1. **Read the story's Interaction Flow and Visual States.** Every criterion must trace to something
   the story actually builds. List the UI patterns it introduces or changes: form, dialog/drawer,
   menu/popover, tabs, table, toast/async feedback, route change, drag interaction, icon-only
   control, custom widget, color-coded status, animation.
2. **Derive criteria per pattern**, using the `accessibility` skill for the specifics. The patterns
   that most often go wrong:
   - **Every control** — reachable and operable by keyboard alone; has an accessible name; focus is
     visible (2.4.7) and not obscured by sticky UI (2.4.11); pointer target at least 24×24px (2.5.8).
   - **Forms** — every input has a programmatic label; errors are tied to their field
     (`aria-describedby`, `aria-invalid`) and on failed submit focus moves to the first invalid
     field or an error summary; required state isn't conveyed by color or `*` alone.
   - **Dialogs / drawers** — focus moves into it on open, stays inside while open, Escape closes it,
     focus returns to the trigger on close, the dialog has a name.
   - **Async feedback** — success/error toasts and "results updated" changes are announced through a
     polite live region without moving focus.
   - **Route changes (SPA)** — document title updates; focus moves to the new page's heading or main.
   - **State shown visually** — status, selection, and validity aren't conveyed by color alone (1.4.1);
     text contrast ≥ 4.5:1, UI components and focus indicators ≥ 3:1 (1.4.3, 1.4.11).
   - **Drag, swipe, or motion** — a single-pointer alternative exists (2.5.7); non-essential animation
     respects `prefers-reduced-motion`.
3. **Write each criterion so a test can check it**, naming the role and accessible name a test would
   locate it by:
   - Weak: "The dialog is accessible."
   - Strong: "After activating *Delete invoice*, focus is inside `dialog` named *Delete INV-001?*;
     Escape closes it and focus returns to the *Delete invoice* button."
4. **Add matching test scenarios** for the story's Test Plan, using role/label locators
   (`getByRole`, `getByLabel`). If the repo already has `@axe-core/playwright` or `jest-axe`, add one
   scan of the new view scoped to the changed region. If it doesn't, don't add the dependency — list
   it under the story's open questions instead.
5. **Name the primitives** from the step above that satisfy each criterion, so implementation reuses
   them instead of rebuilding them.

**Scale to the story.** A story that adds one button gets one or two criteria, not a WCAG checklist.
A story with no new interactive or visual surface (copy change, backend-shaped UI story, CLI output)
gets "No new interactive or visual surface" and nothing else. Padding this section with criteria the
story doesn't exercise is the same defect as scope creep anywhere else.

## Review mode

### Never modify the git staging area

Never run `git stash`, `git reset`, `git checkout -- .`, `git restore`, or anything else that alters
the index or working tree state. Use read-only commands (`git diff --staged`, `git show HEAD:path`).

### Process

1. **Get changes**: `git diff --staged`, or `git diff` if nothing is staged. A caller (e.g.
   `review-pr`) may hand you a diff instead — use that.
2. **Filter to UI surface**: components (`.tsx`, `.jsx`, `.vue`, `.svelte`), templates (`.html`,
   `.heex`, `.erb`, Django/Jinja), stylesheets and utility classes, and client code that handles
   focus, keyboard events, or live regions. If none of these changed, say "No UI surface touched" and
   stop.
3. **Find the scope anchor**: if the work has a user story (`docs/YYYY_MM_DD_*/user-stories.md`),
   read its **Accessibility** section. Every unmet criterion there is a finding.
4. **Rendered check, if one is cheap**: if a dev server for the changed view is already running and a
   browser tool is available (Chrome DevTools MCP preferred, for `lighthouse_audit` and
   `take_snapshot`), run the `accessibility` skill's evidence-led workflow on that view. Don't start
   services the repo's `CLAUDE.md` says to leave alone. Otherwise review statically and say so in one
   line. An automated score is evidence, not conformance — it catches a fraction of real barriers.
5. **Analyze each changed file in full**: names and roles, keyboard operability and focus order,
   focus management on open/close/navigate, form labelling and error association, live-region
   announcements, color and contrast, target size, motion, and semantic structure (landmarks,
   heading order, table headers, lists). For contrast, resolve design tokens to actual values where
   the theme files allow; where they don't (runtime themes), mark the finding *needs rendered check*
   rather than guessing a ratio.
6. **Run the project's a11y linter** (`eslint-plugin-jsx-a11y` or equivalent) on the changed files if
   it's configured.
7. **Report**, in the format below.
8. **Fix**, highest impact first, unless a calling skill owns fixing (see below).

### Fix discipline

- **Native element over ARIA.** A `<div role="button" tabIndex={0} onKeyDown=…>` is fixed by making
  it a `<button>` and deleting the ARIA and key handler, not by patching the handler. Redundant ARIA
  (`role="button"` on a `<button>`, `aria-label` repeating the visible text) is removed, not
  annotated. A net-negative fix is a good fix.
- **Reuse the repo's primitive** over hand-rolling focus traps, live regions, or label wiring.
- **No new dependency** to fix a finding. If one would genuinely help (a focus-trap library, axe in
  tests), recommend it in the report and leave the decision to the user.

### When another skill invokes this one

`enhance-code` and `review-pr` call this skill as a lens. When invoked that way, return findings in
the format below and stop — the caller merges them into its own report and owns fixing (or, for
`review-pr`, not fixing). Don't produce a second, separate report.

## Output format

Group findings by **impact**, not by file. Impact is measured by what a user relying on a keyboard,
screen reader, magnification, or reduced motion can no longer do.

**High Impact** — Must fix. A task in the story is blocked for some users. Examples: a control
unreachable or inoperable by keyboard; a keyboard trap; an input with no label; an icon-only button
with no accessible name; a dialog that doesn't take focus, so screen-reader users never learn it
opened; a form whose errors are neither announced nor associated, so it can't be completed; a
required decision conveyed by color alone.

**Medium Impact** — Should fix. The task is possible but degraded. Examples: text contrast below
4.5:1; focus not returned to the trigger after a dialog closes; an async result that isn't
announced; an SPA route change that leaves focus on the old link; targets under 24×24px; skipped
heading levels; animation that ignores `prefers-reduced-motion`.

**Low Impact** — Nice to have. Examples: redundant ARIA duplicating native semantics; `aria-label`
repeating visible text; decorative images with non-empty `alt`; AAA-only improvements.

For each finding:

1. Category tag — `[A11y: Keyboard]`, `[A11y: Names & Roles]`, `[A11y: Focus]`, `[A11y: Forms]`,
   `[A11y: Announcements]`, `[A11y: Contrast]`, `[A11y: Structure]`, `[A11y: Motion]`,
   `[A11y: Target Size]`, `[A11y: Over-engineering]`
2. File and line in `path:line` format
3. What a user can't do, and the WCAG success criterion it fails
4. The fix, naming the repo primitive to reuse where one exists

## Example output

### High Impact

- **[A11y: Names & Roles] [A11y: Keyboard] `src/invoices/InvoiceRow.tsx:48`** — The row's delete
  control is an `<svg onClick>` with no role, name, or tab stop. Keyboard and screen-reader users
  can't delete an invoice. Fails 2.1.1 Keyboard and 4.1.2 Name, Role, Value.
  - Replace with the project's `IconButton` (`src/ui/IconButton.tsx:12`) and
    `label="Delete invoice INV-001"`.

### Medium Impact

- **[A11y: Focus] `src/invoices/DeleteInvoiceDialog.tsx:30`** — The hand-rolled dialog moves focus
  in on open but never returns it on close, so keyboard users are dropped at the top of the page and
  lose their place in the list. Fails 2.4.3 Focus Order.
  - Rebuild on the project's `Dialog` (`src/ui/Dialog.tsx`, Radix-based), which handles the trap,
    Escape, and focus return. This removes ~40 lines.

### Low Impact

- **[A11y: Over-engineering] `src/invoices/InvoiceFilters.tsx:22`** — `<button role="button"
  aria-label="Apply">Apply</button>`: both attributes restate what the element already provides.
  - Delete `role` and `aria-label`.
