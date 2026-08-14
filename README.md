# my-agentic-workflow

My personal [Claude Code](https://claude.com/claude-code) setup: the skills I've written for my
own development workflow, plus the pointers needed to rebuild the environment on a new machine.

Everything in `skills/` is hand-written. Third-party skills and MCP servers aren't vendored —
they're listed under [Dependencies](#dependencies) with their upstream sources. These are shaped
around how I work (Postgres-heavy backends, React/React Native frontends, PR-driven flow), so take
what's useful and treat the rest as a starting point.

## Install

```bash
git clone https://github.com/<you>/my-agentic-workflow.git
cd my-agentic-workflow
./install.sh
```

This symlinks each directory in `skills/` into `~/.claude/skills/`, so the repo stays the source of
truth — edit a skill in either location and it's the same file, captured by `git` instead of lost
on the next laptop. Then install the [dependencies](#dependencies) the skills you use require.

## The skills

They're built to chain:

```
write-prd  →  write-user-stories  →  story-loop
                                        │
                    per story: test-coverage → enhance-code
                               → review-postgres-schema (if schema touched)
                               → create-pr → review-pr
```

`story-loop` orchestrates the quality gates itself, so running it invokes most of the others. Each
also stands alone — `/enhance-code` on a staged diff is useful without the surrounding machinery.

### Planning

| Skill | What it does |
|---|---|
| `write-prd` | Feature idea → PRD: the why, what, and scope, before any code. |
| `write-user-stories` | PRD → sequenced, implementation-ready stories with test specs, each classified UI or backend. |
| `database-change-modeler` | Models Postgres schema changes, migrations, and indexes as a design exercise — tradeoffs, not implementation. |

### Implementation

| Skill | What it does |
|---|---|
| `story-loop` | The big one. Autonomous loop over a work source (stories doc, PRD, Linear issue, PR findings): research → implement → quality gates → verify → commit → PR → self-review, one item at a time, via sub-agents. Logs decisions and open questions as it goes. |

### Quality gates

| Skill | What it does |
|---|---|
| `test-coverage` | Writes tests for staged changes and audits assertion quality — exact values over vague checks, identity over count, state over status. |
| `enhance-code` | Scans staged code for correctness, safety, security, performance, and over-engineering. Treats *removal* as a first-class fix. |
| `review-postgres-schema` | Reviews staged migrations and queries, delegating to Tiger MCP and Supabase guidance for authority. |
| `review-pr` | Multi-lens PR review against the linked ticket, separating blocking from non-blocking findings. |

### Shipping

| Skill | What it does |
|---|---|
| `create-pr` | Drafts PRs with succinct, thoughtful descriptions. Two modes: Linear-backed, or personal. |
| `record-demo-video` | Records a stakeholder-facing video walkthrough of the user-facing changes on a branch. |

## Dependencies

None of these are bundled. Install what the skills you actually use require.

### MCP servers

| Server | Required by | Where to get it |
|---|---|---|
| **Tiger** | `database-change-modeler`, `review-postgres-schema` | <https://github.com/timescale/tiger-mcp> |
| **Tidewave** | `record-demo-video`; `review-postgres-schema` *(optional — falls back to static analysis)* | <https://tidewave.ai> |
| **Linear** | `create-pr`, `story-loop`, `review-pr` | <https://mcp.linear.app/sse> |

Where a skill mentions an issue tracker generically (`review-pr`, `create-pr`), any equivalent MCP
or CLI works — Jira, GitHub issues, `gh`. Linear is just what I use.

### Third-party skills

One hard dependency: `supabase-postgres-best-practices`, required by `database-change-modeler` and
`review-postgres-schema` — <https://github.com/supabase/agent-skills>, `skills/postgres-best-practices/`.

The rest of what I keep installed, for reference — no skill here needs them:

| Source | Skills |
|---|---|
| [vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills) | `vercel-react-best-practices` (`skills/react-best-practices/`), `vercel-react-native-skills` (`skills/react-native-skills/`), `web-design-guidelines` |
| [vercel-labs/skills](https://github.com/vercel-labs/skills) | `find-skills` |
| [expo/skills](https://github.com/expo/skills) | `building-native-ui`, `native-data-fetching` — both under `plugins/expo-app-design/` |
| [tigrisdata/skills](https://github.com/tigrisdata/skills) | `conventional-commits`, `installing-tigris-storage`, `tigris-bucket-management`, `tigris-object-operations`, `tigris-snapshots-forking` |
| [anthropics/skills](https://github.com/anthropics/skills) | `frontend-design` |
| [nextlevelbuilder/ui-ux-pro-max-skill](https://github.com/nextlevelbuilder/ui-ux-pro-max-skill) | `ui-ux-pro-max` |
| [saadeghi/daisyui](https://github.com/saadeghi/daisyui) | `daisyui` (`skills/daisyui/`) |
| [resend/resend-skills](https://github.com/resend/resend-skills) | `resend` |

### CLI tools

- `git`
- [`gh`](https://cli.github.com) — required by `create-pr` and `review-pr`

## Changelog

[`CHANGELOG.md`](CHANGELOG.md) records what's changed here — skills added, behaviour changed, and
skills retired along with why they're gone.

## License

MIT
