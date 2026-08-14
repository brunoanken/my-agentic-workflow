---
name: create-pr
description: >-
  Create draft PRs with solid, succinct, and thoughtful descriptions.
  Supports two workflows: Linear-backed (official work, references the issue in
  title and description) and personal (no external references, works from git
  history and optional user-stories docs). Use when asked to create a PR, open a
  pull request, or draft a PR description.
---

## Workflow routing

The skill detects which workflow to use based on context:

| Signal | Likely workflow |
|---|---|
| Branch name matches `LIN-\d+` or `lin-\d+` | Linear-backed |
| Linear MCP available + user mentions "Linear" | Linear-backed |
| User mentions a user-stories doc or PRD | Personal (no Linear) |
| None of the above — no Linear MCP, no doc mention | Personal |

**Default**: if ambiguous, ask.

---

## Step 1 — Detect context

Run these checks in parallel to understand the branch and its history:

```bash
git branch --show-current
git log --oneline origin/main..HEAD 2>/dev/null || git log --oneline main..HEAD 2>/dev/null
git diff --stat origin/main..HEAD 2>/dev/null || git diff --stat main..HEAD 2>/dev/null
```

### Linear issue detection

If the branch name contains a Linear ticket pattern (`LIN-\d+` or `lin-\d+`), extract it. Use the Linear MCP `linear_get_issue` to fetch the issue title and description for context.

---

## Step 2 — Ask interactive questions

Always ask these questions before generating. Use the `question` tool — batch them into a single call.

### Common questions (both workflows)

1. **Target branch** — default: `origin/main` (or `main`). Ask to confirm.
2. **Is this a stacked PR?** — i.e., are some asks intentionally deferred to follow-up PRs? If yes, ask what was deferred and why.
3. **Key decisions, tradeoffs, and alternatives considered** — any architectural choices worth calling out? What other approaches were considered and why were they rejected?
4. **Anything specific to highlight** — for the product-log section (non-technical audience).

### Linear-backed workflow extra questions

5. **Linear issue ID** — auto-detected from branch name if possible, then confirm: "I found LIN-XXX from your branch name. Is that the correct issue?" If not found, ask for it.
6. **Any asks from the issue that are NOT included in this PR?** — for the stacked/deferred section.

### Personal workflow extra questions

7. **Is there a user-stories document to use as context?** — look for `USER-STORIES*.md` or `*.user-stories.md` files in the repo. If found, offer to read it. If the doc is uncommitted, read it but **do not reference it** in the PR description (per user preference).

---

## Step 3 — Gather context

### For Linear-backed
- Fetch the issue via `linear_get_issue` with the confirmed issue ID. Extract the title, the key asks from the description, and any discussion of **alternatives considered** (look for comments, design threads, or the issue body mentioning "alternatives", "instead of", "rather than", "considered").
- Read the git diff and commit log for a summary of actual changes.

### For personal
- If the user opted to provide a user-stories doc, read it silently as context. Mine it for **alternatives considered** — look for sections or language discussing what was ruled out and why (do not link or reference the doc itself in the PR).
- Read the git diff and commit log for a summary of actual changes.

---

## Step 4 — Generate the PR

### Title format

**Linear-backed:**
```
[LIN-XXX] Brief description of the change
```
Derive the description from the issue title and the actual work done. Keep it under ~72 characters after the prefix.

**Personal:**
Use a conventional-commit-style summary derived from the actual changes (e.g., `feat: add SLA overdue highlighting to invoice list`).

### Description template — Linear-backed

```markdown
Closes LIN-XXX

## What shipped
<!-- Product-log style bullet points. Non-technical, focused on value. -->
- ...
- ...

## What's included
<!-- Technical changes in this PR. -->
- ...
- ...

## Deferred (stacked PRs)
<!-- Asks from LIN-XXX that are NOT in this PR, and why. -->
- **ASK**: ... — **Why deferred**: ...

## Decisions & tradeoffs
<!-- Key architectural choices made, alternatives considered, and tradeoffs. -->
- **Decision**: ... — **Why**: ... (tradeoff: ...)
- **Alternative considered**: ... — **Why rejected**: ...

### Alternatives considered
<!-- Optional standalone section. Use when there are multiple alternatives worth calling out beyond what fits inline above. Drop if the inline format above covers everything. -->
- **Approach**: ... — **Why we didn't go with it**: ...
```

### Description template — Personal

```markdown
## What shipped
<!-- Product-log style bullet points. Non-technical, focused on value. -->
- ...
- ...

## What's included
<!-- Technical changes in this PR. -->
- ...
- ...

## Deferred
<!-- If stacked, what will be tackled in follow-up PRs. -->
- ...

## Decisions & tradeoffs
- **Decision**: ... — **Why**: ... (tradeoff: ...)
- **Alternative considered**: ... — **Why rejected**: ...

### Alternatives considered
<!-- Optional standalone section. Drop if inline format above covers it. -->
- **Approach**: ... — **Why we didn't go with it**: ...
```

### Product-log guidelines

The "What shipped" section is designed to be copy-pasted into a Slack channel for non-technical stakeholders. Rules:
- Start each bullet with a verb in past tense: "Added", "Fixed", "Improved", "Enabled"
- Describe user-facing value, not implementation details
- No jargon, no internal file paths, no ticket IDs
- If nothing is user-facing yet, describe the groundwork and why it matters

Example:
```
- Added overdue invoice highlighting so finance teams can spot late payments at a glance
- Improved invoice list loading time by 40% for accounts with 1000+ invoices
- Fixed a bug where voided invoices still appeared in payment run suggestions
```

### Technical "What's included" guidelines

- Describe what changed at the code level
- Reference modules, endpoints, models where helpful
- Keep bullets concise — one line per logical change

---

## Step 5 — Create the draft PR

Use `gh pr create`:

```bash
gh pr create \
  --title "..." \
  --body "$(cat /tmp/pr_body.md)" \
  --base main \
  --draft
```

Write the body to a temp file first to avoid shell escaping issues.

**Always create as `--draft`** — the user can mark it ready when they're done reviewing.

---

## Step 6 — Show summary

After creation, display:
- PR URL
- Title
- A copy-pasteable snippet of the "What shipped" section for Slack

---

## Anti-patterns

- **Never reference uncommitted documents** in the PR description or title.
- **Never link Linear issues** in personal-workflow PRs.
- **Never skip the interactive questions** — rushing to generation produces generic PRs.
- **Never merge or mark ready** — the skill creates draft PRs only. The user takes it from there.
- **Don't guess the Linear issue ID** without confirming with the user — branch patterns can be misleading (e.g., a branch for LIN-123 might actually close LIN-124).
- **Don't fabricate product-log bullets** — if there's no user-facing change, be honest about it ("Laying groundwork for..." is fine).
