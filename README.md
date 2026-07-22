# agent-skills

A collection of markdown-only skills for coding agents (Claude Code, Cursor, OpenAI Codex CLI, and other tools that follow the [Agent Skills specification](https://github.com/vercel-labs/skills)).

Each skill is self-contained, portable across tools, and installable individually or all at once via the [`skills`](https://github.com/vercel-labs/skills) CLI.

## Skills in this repo

| Skill | Status | What it does |
| --- | --- | --- |
| [`babysit-pr`](./skills/babysit-pr) | Shipped | Watches an open PR until it's merge-ready — resolves bot review comments, fixes failing CI checks, and loops until all checks are green and no unanswered bot comments remain. |
| [`critique-loop`](./skills/critique-loop) | Shipped | Cross-model critique loop with two entry points. **Full flow:** plan → navigator reviews plan → user approves → implement → navigator reviews diff. **Review-only flow:** skip planning/implementation; navigator adversarially reviews an existing diff. Navigator is pluggable: Codex CLI (default, `gpt-5.5` at xhigh effort), Cursor CLI (`gpt-5.3-codex-xhigh`), or — when driving from OMP — a native `critique-navigator` subagent (`gpt-5.5` at xhigh, no external CLI; agent def ships in `omp/`). All rounds share one navigator session (UUID- or agent-id-tracked) so context persists across phases. Claude fixes code/scope-level asks itself; product/architecture questions are surfaced to the user. Artifacts live in `.critique-loop/` — local-only, auto-added to `.gitignore` so they don't pollute the PR. |
| [`gzplan`](./skills/gzplan) | Draft | Fast single-document plan for a small concrete task — bug fix, refactor, small feature, doc tweak, config change. Produces one focused plan markdown at `.agents/plans/<slug>-plan.md` (gitignored): Task · Context · Approach · Files to touch · Verification · Risks · Open questions. Skips PRD/design/ADR ceremony (that's `gzship`); skips cross-model review (compose with `critique-loop` afterward if you want one). Optional grilling pass when the task description is ambiguous. |
| [`gzreview`](./skills/gzreview) | Draft | Produce a teammate-facing code tour of an existing feature branch. Phase 1 auto-detects the **branch type** (new-feature-with-abstractions, new-feature-on-existing, bug-fix, refactor, performance, config/infra) and picks a matching skeleton; the tour adapts instead of forcing every PR through a feature template. Universal sections — Business value, Scope, Testing strategy, Risks & rollout, Open questions, Upstream artifacts (links to `gzship` docs when present). Diagrams are inline Mermaid fences with theme-safe mid-tone colors. Iterates via `plannotator annotate`; output stays at `.agents/branch-tour.md` (gitignored). |
| [`gzship`](./skills/gzship) | Shipped | End-to-end feature development in 8 phases: BDD scenarios → PRD (Quality Targets / Constraints / Domain Terms / Stakeholders) → architecture & design (current-vs-proposed diagrams in D2 or Mermaid, plus Risks, Integration & Data Ownership) → vertical-slice plan → staged TDD implementation. Lazily produces ADRs under `docs/adr/` with bidirectional links back to the originating feature. Three gates, each requiring both a cross-model navigator review (via `critique-loop`) and a developer review (via the `plannotator` CLI). **Claude Code only** — no `cursor/` or `codex/` variant. |

## Installation

### Recommended: `skills add`

Install any skill into your coding agent of choice. Pick your package-manager runner (all four work identically):

```bash
# Install one skill into one agent (globally)
npx      skills add gzaripov/agent-skills --skill babysit-pr -a claude-code -g
bunx     skills add gzaripov/agent-skills --skill babysit-pr -a claude-code -g
yarn dlx skills add gzaripov/agent-skills --skill babysit-pr -a claude-code -g
pnpm dlx skills add gzaripov/agent-skills --skill babysit-pr -a claude-code -g

# Install one skill into multiple agents
npx skills add gzaripov/agent-skills --skill babysit-pr -a claude-code -a cursor -a codex -g

# Install all skills at once
npx skills add gzaripov/agent-skills --all -a claude-code -g

# List what's in this repo without installing
npx skills add gzaripov/agent-skills --list
```

Swap `-g` (global) for the default (project-local). See `npx skills add --help` for all flags.

### Manual install

Each skill directory has its own per-tool layout:

```
skills/<name>/
├── SKILL.md               # Claude Code (frontmatter + workflow)
├── cursor/<name>.mdc      # Cursor rule (alwaysApply: false)
└── codex/<name>.md        # Codex CLI prompt
```

Install the one you need into your tool's native extension path. See each skill's own README or the file comments for per-tool instructions.

## Why markdown-only?

Every skill here is plain markdown plus a few shell commands. No Node.js runtime, no MCP servers to configure, no native binaries, no Docker. This means:

- **Portable** — works in any tool that reads markdown and shells out: Claude Code, Cursor, Codex CLI, Trae, Antigravity, Aider, and so on.
- **Debuggable** — open the file, read exactly what the agent was told.
- **Forkable** — copy any skill, adapt it, commit it alongside your project.
- **No install surprises** — no `tmux raw-mode` native bugs, no permission scripts failing on a fresh install. If your agent can read a file and run `git`, it can run these skills.

Philosophy borrowed from [`wanshuiyin/ARIS`](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep); domain here is general software engineering.

## Updating & removing

```bash
# Update all installed skills from this repo
npx skills update gzaripov/agent-skills

# See what's currently installed (any runner)
npx skills list

# Remove one
npx skills remove gzaripov/agent-skills --skill babysit-pr
```

## Contributing

Each skill lives in its own subdirectory under `skills/`. To add a new one:

1. Create `skills/<name>/SKILL.md` with valid frontmatter (`name`, `description`, optional `license`, `allowed-tools`).
2. Optionally add `cursor/<name>.mdc` and `codex/<name>.md` variants.
3. Add a row to the table at the top of this README.
4. Open a PR.

## License

MIT. See [LICENSE](./LICENSE).
