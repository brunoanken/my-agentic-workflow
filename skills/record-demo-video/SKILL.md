---
name: record-demo-video
description: Record a stakeholder-facing video walkthrough of the user-facing changes introduced in the current branch/PR, using Tidewave's browser video recording. Use when the user asks to record, film, or demo a video of recent changes, a feature, or "what's new" for stakeholders.
---

# Record Demo Video

Produce a short, narrated video that shows stakeholders what changed in the app — new things users can do, or how an existing flow now behaves differently. This is a highlight reel, not a QA recording: only show what a non-technical viewer would actually care about.

## When to Use

- "Record a video of what we shipped in this branch/PR"
- "Make a demo of the new [feature]"
- "Show stakeholders what changed"

## Process

### 1. Determine scope

Default: diff the current branch against `main` (`git diff main...HEAD` / `git log main..HEAD`) to capture everything introduced on the branch, whether committed or still staged — this matches what the PR will contain.

Fall back to asking the user instead of guessing when:
- You're already on `main`/`master` (no branch diff to compute)
- The diff is empty but there are staged/unstaged changes (use those instead, confirm with the user)
- The user mentions a PR number, ticket, or a specific feature — use that as the scope instead of the raw diff
- The diff spans many unrelated features — ask which one(s) to cover, rather than cramming everything into one video

### 2. Turn the diff into a narrative, not a file list

Read the diff and identify only **user-facing** changes: new pages/flows, new controls, changed behavior, fixed bugs a user would have noticed. Ignore refactors, internal-only changes, test files, and backend-only work with no visible effect.

Group related changes into a short list of "beats" — the scenes the video will show, in a logical order (e.g. entry point → new capability → result). Prefer one cohesive walkthrough over many disconnected clips, same as this repo's E2E testing philosophy: a few high-value flows beat many tiny ones.

If nothing in scope is user-visible, tell the user there's nothing worth recording instead of forcing a video.

For each beat, write:
- The page/route to be on and the starting state
- The action(s) to perform
- A short overlay line (present tense, benefit-focused: "You can now filter units by status" — not "Added a status filter prop")

### 3. Rehearse before recording

Recording is expensive to redo, and `browser.snapshot()` references change between calls. Before touching `startRecording`:
- Walk the full flow once with plain `browser_eval` (no recording)
- Resolve stable selectors (labels, roles, test ids) for every step — don't rely on snapshot refs that may shift
- Confirm any data you need exists (seed data, an org/property/lease to point at) — create it via the UI first if it doesn't
- Note anything flaky (loading states, animations, toasts) and how to wait for it reliably

Once the rehearsal works end-to-end, reset to a clean starting state (reload, log out/in, navigate back) before recording — the recording should not show your rehearsal debris.

### 4. Record

In a single `browser_eval` call when possible:
1. `await browser.startRecording({ description: "..." })` — describe the overall video
2. For each beat: `await browser.overlay("...")` with the line drafted in step 2, then perform the action using `browser.click`/`browser.fill` (these animate) rather than raw DOM manipulation
3. `await browser.stopRecording()` at the end

Do not use `browser.zoom` — keep the recording at a fixed, un-zoomed viewport throughout.

If something goes wrong mid-recording, `await browser.abortRecording()` and fix the flow before retrying — don't ship a recording with visible mistakes.

Keep the total flow well under 5 minutes (recording auto-stops there). If scope is large, prioritize the highest-value changes and tell the user what was left out rather than silently cramming everything in.

### 5. Wrap up

Report where the video was saved and give a one-line summary of what it covers. If you skipped anything from the original scope (low-value internal changes, or beats cut for time), say so.

## Anti-patterns to Avoid

- Recording internal/technical changes (refactors, type changes, non-visible fixes)
- Long silent stretches with no overlay narration
- Skipping the rehearsal and burning recording attempts on selector trial-and-error
- One video per tiny change instead of one cohesive walkthrough per session/branch
