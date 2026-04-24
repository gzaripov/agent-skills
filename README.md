# agent-skills

A collection of markdown-only skills for coding agents (Claude Code, Cursor, OpenAI Codex CLI, and other tools that follow the [Agent Skills specification](https://github.com/vercel-labs/skills)).

Each skill is self-contained, portable across tools, and installable individually or all at once via the [`skills`](https://github.com/vercel-labs/skills) CLI.

## Skills in this repo

| Skill | Status | What it does |
| --- | --- | --- |
| [`babysit-pr`](./skills/babysit-pr) | Shipped | Watches an open PR until it's merge-ready — resolves bot review comments, fixes failing CI checks, and loops until all checks are green and no unanswered bot comments remain. |
| [`pair`](./skills/pair) | Planning | XP-style cross-model pair-programming skill. One agent drives (plan + implement), a different-model agent navigates (adversarial plan review + code review). Currently in design — see [`skills/pair/PLAN.md`](./skills/pair/PLAN.md). |

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
