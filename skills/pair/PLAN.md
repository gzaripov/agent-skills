# Plan — XP-style cross-model pair skill (new public repo)

## Context

Build a new skill, shipped as a separate public GitHub repo under `gzaripov/<name-tbd>`, that implements an XP pair-programming workflow across **two different coding agents**: one drives (plans + implements), the other navigates (reviews). The four-phase flow the user described is:

1. Driver writes a plan.
2. Navigator (other model) reviews the plan.
3. Driver implements the plan.
4. Navigator reviews the diff.

**Why this is worth building even given prior art.** Research surfaced several overlapping tools — OpenAI's official `codex-plugin-cc`, `ching-kuo/claude-codex`, `JuliusBrussee/cavekit`, `wanshuiyin/ARIS`, `religa/multi_mcp`, `praneybehl/code-review-mcp`. All of them are MCP-based and all but ARIS are one-directional (Claude → Codex). The research literature (Microsoft Critique Mode: +13.8% DRACO; Reflection: HumanEval 80% → 91%) validates the pattern; the gap is in the *packaging*. Our four differentiation angles (all confirmed by the user):

- **Bidirectional** — either tool can be the driver. Symmetric SKILL.md / cursor / codex variants.
- **Shared-file handoff, no MCP** — all state lives in `.pair/*.md`, driver invokes navigator via headless CLI (`codex exec`, `claude -p`). Works in any tool that can read/write files and shell out, no per-user MCP setup.
- **Multi-host from day one** — same `babysit-pr` pattern: install targets for Claude Code, Cursor, and Codex CLI.
- **XP vocabulary** — *driver* / *navigator*, with explicit role-swap guidance. No prior art frames it this way.

**Intended outcome.** A user on any of the three host tools can say "/<skill> on this task" and get a plan→review→implement→review loop with an adversarial cross-model critic, without installing MCP servers. Works with whichever other CLI they already have (or can install).

## User decisions already captured

| Decision | Choice |
| --- | --- |
| Direction | Bidirectional (Claude drives + Codex drives + Cursor drives) |
| Handoff mechanism | Shared files in `.pair/` + headless CLI invocations |
| Loop depth | Loop until navigator approves, no cap (user can always interrupt) |
| Global context file | No AGENTS.md template — SKILL.md is self-sufficient |
| Artifact lifecycle | `.pair/*.md` committed to git (full audit trail in the PR) |
| Name | **Deferred** — user will pick after reviewing this plan. Candidates: `pair`, `pair-program`, `ping-pong`, `driver-navigator`, `xp-pair`, `pair-agents`. I'll default to `pair` in the plan below as a placeholder. |

## Repo layout

```
<name>/
├── README.md                   # install, usage, prerequisites, FAQ
├── LICENSE                     # MIT
├── SKILL.md                    # Claude Code driver (invokes codex exec OR claude -p)
├── cursor/
│   └── <name>.mdc              # Cursor driver (invokes codex exec)
└── codex/
    └── <name>.md               # Codex driver (invokes claude -p)
```

No MCP config, no examples dir, no AGENTS.md. Each variant file is self-contained.

## Runtime artifact layout (inside a user's project)

Per PR / task, the driver creates:

```
.pair/
├── <slug>-plan.md              # driver writes in Phase 1
├── <slug>-plan-review.md       # navigator writes in Phase 2
├── <slug>-plan-review-2.md     # second round if CHANGES_REQUESTED (numbered)
├── <slug>-diff.md              # driver writes in Phase 3 (git diff + summary)
└── <slug>-code-review.md       # navigator writes in Phase 4
```

`<slug>` = branch name or short task identifier. Files are committed with `chore(pair): …` commits; verdict lines at the bottom of each review file (`VERDICT: APPROVE | CHANGES_REQUESTED | BLOCK`).

## Workflow (encoded in all three variant files)

### Phase 1 — Plan (driver)

- Driver reads the task, explores the repo, drafts `.pair/<slug>-plan.md` with sections: *Context*, *Approach*, *Files to modify (with paths)*, *Verification*, *Open questions*.
- Driver stages and commits: `chore(pair): plan for <slug>`.

### Phase 2 — Plan review (navigator)

- Driver invokes the navigator CLI. Two concrete shapes, selected by what's installed:
  - **Claude-driver → Codex-navigator:** `codex exec --model gpt-5 "$PROMPT"`
  - **Codex-driver → Claude-navigator:** `claude -p "$PROMPT"`
  - **Cursor-driver → either:** shell out to whichever CLI is present.
- The prompt template (stored inline in each SKILL file):
  > You are the navigator in an XP pair-programming session. Read `.pair/<slug>-plan.md` and the referenced code. Be adversarial — probe for missing edge cases, risky assumptions, unstated dependencies, and simpler alternatives. Write your review to `.pair/<slug>-plan-review.md`. End with exactly one line: `VERDICT: APPROVE` or `VERDICT: CHANGES_REQUESTED` (with numbered requested changes) or `VERDICT: BLOCK` (with rationale).
- Driver reads the verdict:
  - `APPROVE` → Phase 3.
  - `CHANGES_REQUESTED` → driver revises plan, commits `chore(pair): revise plan (round N)`, **re-invokes** navigator; loop until approve.
  - `BLOCK` → driver surfaces the rationale to the user; stop.

### Phase 3 — Implement (driver)

- Driver implements exactly what the approved plan describes.
- Uses conventional commits (`feat:`, `fix:`, etc.).
- After implementation: runs the project's lint + tests.
- Driver writes `.pair/<slug>-diff.md` with: the commit range (`<plan-sha>..HEAD`), a short what-changed summary, and pointers to the files touched.
- Commits: `chore(pair): implement <slug>`.

### Phase 4 — Code review (navigator)

- Driver invokes navigator CLI with:
  > You are the navigator. The approved plan is at `.pair/<slug>-plan.md`. The implementation is at commits `<plan-sha>..HEAD` (summary in `.pair/<slug>-diff.md`). Run `git diff <plan-sha>..HEAD`, read the changed files, and verify: (a) the implementation matches the approved plan, (b) no scope creep, (c) no introduced bugs, (d) tests cover the changes. Write to `.pair/<slug>-code-review.md`. End with `VERDICT: APPROVE` or `VERDICT: CHANGES_REQUESTED` (with line-referenced asks).
- Driver reads verdict:
  - `APPROVE` → session complete; summary report; user merges at will.
  - `CHANGES_REQUESTED` → driver fixes, commits, re-invokes navigator; loop.

### Role swap (XP ping-pong variant)

One section in each SKILL file documents a *swap* mode the user can opt into: after Phase 4 APPROVE, the roles flip for the next task (yesterday's navigator becomes today's driver). Not enforced by the skill — just documented as a workflow.

## Files to create (concrete list for implementation)

| Path | Purpose |
| --- | --- |
| `/Users/gzaripov/code/<name>/README.md` | Install (manual + `skills add`), usage, prerequisites, differentiation section citing prior art. Mirror `babysit-pr` README structure. |
| `/Users/gzaripov/code/<name>/LICENSE` | MIT, copyright 2026 Grigory Zaripov. |
| `/Users/gzaripov/code/<name>/SKILL.md` | Claude Code skill with frontmatter (`name`, `description`, `license`, `allowed-tools` including `Bash(codex exec *)`, `Bash(claude -p *)`, `Bash(git *)`) and the four-phase workflow body. |
| `/Users/gzaripov/code/<name>/cursor/<name>.mdc` | Cursor rule variant, same body, Cursor frontmatter (`alwaysApply: false`). |
| `/Users/gzaripov/code/<name>/codex/<name>.md` | Codex prompt variant, same body, no frontmatter. |

## Patterns to reuse from `babysit-pr`

- README structure: hero paragraph → What it does → Prerequisites → Recommended `skills add` (npx/bunx/yarn dlx/pnpm dlx) → Manual install (per-host) → Updating → Uninstalling → Stopping rules.
- Multi-runner package-manager matrix.
- Stopping rules section at the bottom of each SKILL variant.
- `Bash(...)` allowlist frontmatter tuned to the actual commands used.

## Prior art — deep dive

### `openai/codex-plugin-cc` (official)

- **What it does.** Installs Codex CLI as an MCP plugin inside Claude Code. Exposes `/codex:review`, `/codex:adversarial-review`, `/codex:rescue` (delegate task to Codex), `/codex:status`, `/codex:result`, `/codex:cancel`. `--base <ref>` reviews branches; `--wait`/`--background` for sync/async.
- **Workflow.** Review-centric, not plan-implement-review. Claude drives; Codex is invoked on demand to audit, challenge, or take over a task.
- **Direction.** One-way (Claude → Codex only). Codex cannot initiate actions in Claude Code.
- **Hosts.** Claude Code only.
- **Setup.** Node 18.18+, global `@openai/codex`, ChatGPT subscription or OpenAI API key, `codex login`, `/codex:setup`.
- **Strengths.** Official, well-maintained, zero-friction for existing Claude Code users.
- **Limitations for our use case.** Claude-only, one-way, no plan phase, no Cursor support, MCP-required.

### `ching-kuo/claude-codex`

- **What it does.** Five slash commands: `/plan-codex`, `/claude-codex`, `/execute-codex`, `/tdd-claude-codex`, `/tdd-execute-codex`. Plan saved to `.claude/plan/<feature>.md`; Codex audits (max 3 rounds). Implementation uses size-based smart routing: small changes (≤2 files, ≤30 lines) → Claude; larger → Codex.
- **Workflow.** Closest to ours: plan → plan audit (3 rounds max) → implement → code review (3 rounds max).
- **Handoff.** Files for the plan (`.claude/plan/<feature>.md`), MCP for Codex's git diff fetch.
- **Direction.** One-way (Claude → Codex).
- **Hosts.** Claude Code only.
- **Strengths.** Clean four-phase flow, bounded loop, TDD variant.
- **Limitations for our use case.** Claude-only, one-way, MCP dependency, no XP vocabulary, no role swap.

### `JuliusBrussee/cavekit` — deep dive (user asked for specifics)

- **What it is (end-to-end).** A spec-driven development framework for Claude Code, with optional Codex integration. Four commands run sequentially:
  1. **`/ck:sketch`** — decomposes requirements into *kits* with R-numbered specs and testable acceptance criteria.
  2. **`/ck:map`** — generates a tiered dependency graph and coverage matrix from kits.
  3. **`/ck:make`** — autonomous parallel build loop: groups independent tasks into waves, dispatches concurrent subagents, validates each tier before advancing.
  4. **`/ck:check`** — gap analysis + peer review (with Codex if available); returns APPROVE/REVISE/REJECT.
  Ships extra commands: `/ck:judge`, `/ck:converge` (proposed), runtime hooks, stop-hook automation, token ledger, task registry. Node.js orchestration under the hood.
- **Core philosophy.** *"The spec is the product. The code is a derivative."* Every line of code traces to a requirement; every requirement has acceptance criteria. Claimed USP: spec-first traceability + automated parallelization + validation gates.
- **Ecosystem.** Part of a trio: cavekit (builds), `caveman` (compresses what the agent says), `cavemem` (compresses what the agent remembers).
- **Cross-model review.** Optional. Codex adds a design-challenge gate before build, tier gates between waves, and a command-safety classifier. Without Codex: "design challenge skipped, tier gate skipped, command gate falls back to static allowlist. Cavekit works the same."
- **Hosts.** Claude Code (primary — slash commands + subagents), Codex (secondary — skills + plugin bundles). **No Cursor support.**

#### Traction and community signal (as of 2026-04-20)

- **599 stars, 44 forks** on the repo (`JuliusBrussee/cavekit`), created 2026-03-14 — ~5 weeks old. Strong early traction.
- Active community PRs landing: Linux/WSL tmux fix (PR #20), OpenCode portable port (PR #21), codex-review.sh CLI update (PR #23). Project author is shipping (autonomous runtime layer PR #17 merged recently).

#### Weak points / pain points (from open issues and PRs)

1. **Linux install breakage (#22, PR #20).** `install.sh` fails on Linux due to undefined terminal constants `unix.TIOCGETA/TIOCSETA` in the raw-mode tmux handling. This is exactly the kind of fragility that comes with heavy native tooling — our markdown-only design has zero surface area for this class of bug.
2. **Permission errors on fresh install (#9, PR #10, PR #16).** `/ck:judge` and `/ck:make` hit permission issues for new users; multiple follow-up fixes required. Indicates the setup story is complex.
3. **Reviewer lock-in to Codex (#13).** Community asking for OpenCode as an alternative adversarial reviewer when Codex unavailable. Our design's "any CLI" approach answers this out of the box.
4. **No Copilot CLI integration (#14).** Feature request sitting open. Again, "any CLI" beats "Codex or nothing."
5. **Loop tightness (#24, proposed `/ck:converge`).** Users want a fully-autonomous `make → check → make` loop until approval. Cavekit's current review isn't tight enough as an iterate-until-green loop.
6. **Framework weight.** Users opting in get subagent orchestration, runtime hooks, a stop-hook engine, budgets, a token ledger, and a task registry. Great for committed users; high perceived setup cost for someone who just wants "two models double-check each other."
7. **Cursor users are locked out.** The fastest-growing agent host is not a supported target.

### `wanshuiyin/ARIS`

- **What it is.** Markdown-only, framework-free skills for autonomous ML research. 31+ skills across four workflows: idea discovery, implementation, iterative review, paper writing (+ rebuttal).
- **Cross-model review.** Core. `auto-review-loop` orchestrates GPT-5.4 xhigh via Codex MCP as an adversarial reviewer. Configurable difficulty: `medium` (standard), `hard` (reviewer memory across rounds), `nightmare` (reviewer reads codebase directly via `codex exec`). Optional Oracle-Pro routing.
- **Hosts.** Claude Code + Codex CLI + Cursor + Trae + Antigravity + anything that reads markdown + shells out.
- **Philosophy.** *"The entire system is plain Markdown files. No framework to learn, no database to maintain, no Docker to configure."*
- **Limitations for our use case.** All skills are ML-research-specific (LaTeX, GPUs, ICLR/NeurIPS). Community extensions exist for Communications, EDA, Robotics, Systems research — but no general software engineering track.

### `religa/multi_mcp`, `praneybehl/code-review-mcp`

- MCP servers orchestrating multiple models (OpenAI + Anthropic + Google) for consensus code review. Security-focused (OWASP Top 10 scans). MCP-dependent, Claude Code only.

### `agents.md`

- Open standard (Linux Foundation's Agentic AI Foundation). A markdown file at repo root for cross-tool project context. Supported by Codex, Cursor, Copilot, Windsurf, Factory. Complementary, not a competitor.

## Options analysis (detailed)

### Option A — Standalone repo, SWE-focused

- **What.** New repo under `gzaripov/<name>`. Same philosophy as ARIS (markdown-only, multi-host, bidirectional, adversarial) but specialized for general software engineering. XP pair-programming framing. Three host variants: Claude Code (SKILL.md), Cursor (`.mdc`), Codex (prompt).
- **Scope.** Four SKILL files + README + LICENSE. No Node.js, no subagents, no hooks, no token ledger. Just plan → review → implement → review using `.pair/*.md` and `codex exec` / `claude -p` shell-outs.
- **Pros.**
  - Clean slate; no legacy from ARIS's research branding.
  - Full control over naming, framing (XP vocabulary), and UX.
  - Own the audit trail of design decisions.
  - Immediately installable via existing `npx skills add` (already validated with `babysit-pr`).
- **Cons.**
  - Another entry in an already crowded landscape.
  - No built-in audience — discovery relies on README + word of mouth.
  - Duplicates some ARIS concepts (auto-review-loop verdict format, CLI-agnostic reviewer).
- **Time to ship.** Low — 3 SKILL files + README, maybe half a day.
- **Maintenance.** Low — markdown, no runtime.

### Option B — Contribute a software-engineering track upstream to ARIS

- **What.** Open PR(s) to `wanshuiyin/ARIS` adding skills under a new `software-engineering/` or `swe/` workflow: `pair-plan`, `pair-review-plan`, `pair-implement`, `pair-review-code`. Reuse ARIS's existing `auto-review-loop` machinery where possible. Follow the existing skill conventions.
- **Pros.**
  - Piggyback on ARIS's audience (stars, existing users, already in find-skills listings).
  - No new repo to maintain.
  - Strengthens an existing ecosystem instead of fragmenting it.
  - ARIS already supports all three of our target hosts.
- **Cons.**
  - Your skills live under someone else's brand and release cadence.
  - Need to align with ARIS's conventions (difficulty levels `medium`/`hard`/`nightmare`, reviewer dispatch patterns).
  - PR might not be accepted — ARIS is explicitly branded as *research*, not general SWE. Worst case: work done, merged nowhere.
  - Your naming (XP / driver / navigator) may clash with ARIS's existing vocabulary.
- **Time to ship.** Medium — need to study ARIS's conventions, match style, review cycle with maintainer.
- **Maintenance.** Depends on whether merged. If not, fork (see Option C). If yes, ongoing PRs.

### Option C — Fork ARIS, prune to SWE-only

- **What.** Hard-fork `wanshuiyin/ARIS`, delete all ML-research skills, keep `auto-review-loop` machinery and philosophy, rebuild workflows around XP pair programming.
- **Pros.**
  - Fastest to a working artifact — inherit the whole review-loop scaffolding.
  - Preserve proven patterns (difficulty levels, verdict formats, multi-reviewer dispatch).
  - Standalone — own release cadence.
- **Cons.**
  - Fork stigma and attribution burden (license compliance, crediting upstream).
  - Carry forward ARIS's abstractions and vocabulary, which may not fit XP framing cleanly.
  - Upstream changes drift quickly — either you keep syncing or accept divergence.
  - 31 skills to prune and reshape is more work than writing 4 from scratch.
- **Time to ship.** Medium — prune, rename, rewrap; probably a day or two.
- **Maintenance.** High if you sync upstream; medium if you fully diverge.

### Option D — Don't build; recommend ARIS + `auto-review-loop` to users

- **What.** Publish a short blog post / gist: "Here's how to use ARIS's auto-review-loop for general SWE." Contribute an example config. Stop there.
- **Pros.**
  - Zero code written. Zero maintenance.
  - Honest — if the tooling is already there, the ethical move is to point at it.
- **Cons.**
  - The domain gap is real: ARIS skills reference LaTeX, GPU experiments, conference venues. Users would need to mentally translate every instruction.
  - No XP framing (which you've asked for).
  - No bidirectional symmetric variants.
  - Missing a skill-author's natural next step after `babysit-pr`.
- **Time to ship.** ~1 hour.
- **Maintenance.** None.

## Recommendation

**Option A (standalone repo, SWE-focused).** Reasons, ranked:

1. **Cavekit's weak points validate the design.** Three of the top user complaints (Linux install breakage, Codex lock-in, Copilot CLI missing) are non-issues for a markdown-only + shell-out architecture. Zero native code, any CLI works.
2. **Cursor is a real gap.** None of codex-plugin-cc, claude-codex, cavekit, multi_mcp support Cursor. ARIS does — but only for ML.
3. **Bidirectional + XP framing is uncovered.** All five SWE-oriented prior-art projects treat Codex as *the reviewer* permanently. None document a role swap.
4. **Low cost.** 4 files, half a day. Even if it gets 5 stars, it cost nothing.
5. **ARIS upstream contribution (Option B) is a worse bet.** ARIS is explicitly research-branded; a general-SWE track contributes to, rather than inherits, maintainer attention. High risk of "cool, but doesn't fit here."

## ARIS `auto-review-loop` conventions (fetched) — what to borrow, what not to

ARIS's convention actually differs from what we'd want. Summary:

| Aspect | ARIS `auto-review-loop` | Our design |
| --- | --- | --- |
| Verdict | Numeric score (1–10) + categorical (`ready`/`almost`/`not ready`); stop when `score ≥ 6 AND verdict ∈ {ready, almost}` | `VERDICT: APPROVE / CHANGES_REQUESTED / BLOCK` (matches GitHub PR review vocabulary, fits XP pair framing) |
| State dir | `review-stage/` | `.pair/` |
| Files | `AUTO_REVIEW.md` (append-log), `REVIEW_STATE.json`, `REVIEWER_MEMORY.md`, `findings.md` | `<slug>-plan.md`, `<slug>-plan-review.md`, `<slug>-diff.md`, `<slug>-code-review.md` — no JSON state, no separate memory file |
| Invocation | MCP primary (`mcp__codex__codex` + `codex-reply` for continuity); `codex exec` fallback for "nightmare" mode | Shell-out primary (`codex exec` / `claude -p`); no MCP dependency |
| Round cap | `MAX_ROUNDS = 4`; on max, document blockers and suggest pivot | Uncapped (per user decision); user can interrupt |

**Decision: don't adopt ARIS's conventions.** Rationale:

- **Verdict format:** `APPROVE/CHANGES_REQUESTED/BLOCK` maps 1:1 to GitHub PR review verbs users already know. ARIS's score+categorical is more nuanced but requires interpretation. XP framing favors the PR-review vocabulary.
- **State dir:** `.pair/` matches skill name and XP framing; `review-stage/` suggests a research pipeline.
- **No JSON state:** markdown-only is our philosophical commitment; presence/absence of files encodes enough state.
- **Invocation:** shell-out is the *differentiator* vs. every other SWE-oriented prior-art tool. Keeping MCP optional (documented, not required) preserves the no-setup win.
- **Round cap:** user explicitly asked for "no cap." We add a safety note in the stopping-rules section ("ask the user after N rounds if nothing is converging") but don't enforce.

**What we do credit to ARIS in the README:** the philosophy — "plain markdown, no framework, works in any tool that reads files and shells out." That's their legitimate contribution and we should point to it.

## Final recommendation — ready for approval

Option A (standalone repo). Concrete execution sequence when plan is approved:

1. **User picks a name** from `pair`, `pair-program`, `ping-pong`, `driver-navigator`, `xp-pair`, `pair-agents`, or a fresh suggestion.
2. `mkdir /Users/gzaripov/code/<name>` and write four files: `README.md`, `LICENSE`, `SKILL.md`, `cursor/<name>.mdc`, `codex/<name>.md`.
3. `git init -b main`; `gh repo create gzaripov/<name> --public --source=. --remote=origin --description "XP pair-programming skill …" --push`.
4. Install locally into `~/.claude/skills/<name>/` and smoke-test per the Verification section above.

The only blocker now is the naming decision.

## Out of scope

- MCP server implementation (explicitly rejected in favor of shared files).
- Shipping an `AGENTS.md` template.
- Automated model-router logic (the driver picks which CLI to shell out to; the skill documents how).
- Cost/latency telemetry.
- Merging the PR at the end (same boundary as `babysit-pr`).

## Execution steps (after plan approval)

1. Resolve the naming question with the user.
2. `mkdir /Users/gzaripov/code/<name>` and write `README.md`, `LICENSE`, `SKILL.md`, `cursor/<name>.mdc`, `codex/<name>.md`.
3. `git init -b main`, commit, `gh repo create gzaripov/<name> --public --source=. --remote=origin --push`.
4. Install locally into `~/.claude/skills/<name>/` for smoke-testing.

## Verification

Dry-run end-to-end inside a throwaway git repo:

1. `mkdir /tmp/pair-test && cd /tmp/pair-test && git init && gh repo create pair-test --private --source=. --push`.
2. Install the skill into Claude Code (user-level), create a branch, open a trivial PR.
3. Invoke the skill on a small task (e.g., "add a `hello()` function to `src/index.js` with one unit test").
4. Confirm:
   - `.pair/<slug>-plan.md` is created and committed.
   - `codex exec` is invoked (or the reverse if Codex is driving) and writes `.pair/<slug>-plan-review.md` with a `VERDICT:` line.
   - The driver loops on `CHANGES_REQUESTED` and proceeds on `APPROVE`.
   - Implementation commits land.
   - `.pair/<slug>-code-review.md` is produced.
5. Repeat with Codex as driver (`codex/<name>.md` prompt) invoking `claude -p` as navigator.
6. Repeat with Cursor as driver.
7. Verify the prompt text survives through all three shells without quoting bugs.

Manual QA checklist lives at the bottom of the README.

## Open questions for the user (pre-execution)

- **Name** — pick from: `pair`, `pair-program`, `ping-pong`, `driver-navigator`, `xp-pair`, `pair-agents`, or a fresh suggestion.
- **Default navigator model** — should SKILL.md hardcode `--model gpt-5` for `codex exec`, or leave model selection to the user's codex config?
