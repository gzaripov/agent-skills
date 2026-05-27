---
name: gzreview
description: Use when producing a teammate-facing walk-through of an existing feature branch — a story-first code tour that opens with business value, adapts to the branch type (new feature, fix, refactor, perf, config), and shows where the change is used. Renders inline Mermaid diagrams (theme-safe mid-tone colors) and iterates via `plannotator annotate`. Triggers — "tour this branch", "code-tour the feature", "write a walk-through of the diff", "explain this PR to the team", "review my branch", "produce a reviewer-facing summary of these changes".
license: MIT
compatibility: Claude Code only — uses subagents (the Agent tool) for parallel branch exploration and task tracking. Requires the `plannotator` CLI installed (`plannotator --help`) for the iteration loop. Mermaid CLI is invoked on-demand via `bunx @mermaid-js/mermaid-cli` to verify rendered diagrams (no install needed). Run from inside a git repository, on a feature branch with commits ahead of `main`/`master`.
allowed-tools: Bash(git log *) Bash(git diff *) Bash(git status *) Bash(git rev-parse *) Bash(git branch --show-current) Bash(grep *) Bash(rg *) Bash(find *) Bash(ls *) Bash(mkdir -p .agents/*) Bash(bunx @mermaid-js/mermaid-cli*) Bash(qlmanage *) Bash(plannotator --help) Bash(plannotator annotate *) Bash(git add .gitignore) Bash(git commit -m *) Agent
---

Produce a teammate-facing code tour of an existing feature branch and iterate it via Plannotator. The output is one markdown document a colleague can read cold and come away understanding *why* the branch exists, *what changed* (adapted to the branch type), and *where it's used*. Diagrams are inline Mermaid fences — no external SVGs to break path resolution.

## Prerequisites

- The working directory is a git repository with commits ahead of `main`/`master`.
- The `plannotator` CLI is installed and runnable (`plannotator --help`) — it runs the developer-review iteration loop.
- Network access for `bunx @mermaid-js/mermaid-cli` (one-time download per shell).
- macOS for the visual-verification step (`qlmanage` converts SVG → PNG so the rendered diagram can be inspected; on Linux, substitute `rsvg-convert` or `inkscape`).

## Core principle

**A tour is not an inventory.** A list of "Change 1, Change 2, Change 3…" is what the diff already shows. A tour is the narrative a developer would speak to a teammate in front of the diff: *"Here's why we did this, here's what changed, here's where it's used."*

If your draft reads like a changelog, it's wrong. Rewrite it as a story.

## Output

One markdown file at `.agents/branch-tour.md` (or a path the user requests). The skill creates the file, iterates it, and leaves it as the deliverable. **`.agents/` stays gitignored** — tour markdowns are working artifacts the developer hands off to teammates (PR description, design doc, internal wiki, Slack), not files committed to the repo's history. Diagrams live in inline ` ```mermaid ` fences inside the same file — no separate SVG / PNG artifacts in the deliverable path.

## The 5 phases

### Phase 0 — Frame & track

Confirm the working directory is a git repo on a feature branch (not `main`/`master`). Derive `<base-branch>` (default `main`; fall back to `master`). Create `.agents/` if it doesn't exist.

**Gitignore the working directory.** Tour markdowns and diagram scratch never land in the repo. On first run, ensure `.agents/` is gitignored:

```bash
if ! grep -qxE '\.agents/?' .gitignore 2>/dev/null; then
  echo ".agents/" >> .gitignore
  git add .gitignore
  git commit -m "chore: gitignore .agents working directory"
fi
```

The `.gitignore` commit is the only repo-history change `gzreview` ever makes. Everything else is local.

Open a Task/Todo list with one entry per phase — the developer watches this list.

### Phase 1 — Explore (inventory + branch type + abstractions)

**REQUIRED BACKGROUND:** `references/structure.md` — read before exploring; it defines the six branch types and their signals.

Three sub-steps, executed in one fan-out:

**(a) Inventory the diff.** Run `git log <base>..HEAD --oneline` and `git diff <base>..HEAD --stat`. From the file list, identify new files, files with large positive deltas, test files / specs, and config/infra files. Read commit messages in chronological order.

**(b) Detect the branch type.** From the inventory signals, classify the branch as one of:

- `new-feature-with-abstractions` — several new files, new type/schema/interface declarations, new tests, `feat:` commits
- `new-feature-on-existing` — few new files, one module touched heavily, new tests for new behavior, `feat:` commits, no new types
- `bug-fix` — tiny diff in one or two files, new regression test or modified test, `fix:` commits, defect cited
- `refactor` — files moved/renamed, test deltas show preservation not new behavior, no new types, `refactor:` commits
- `performance` — targeted changes in hot paths, benchmark/perf deltas, `perf:` commits
- `config / infra` — `package.json` / lockfiles / Dockerfile / CI config / `tsconfig.json` changes, `chore:` / `build:` / `ci:` commits

**If the signals are ambiguous** (e.g. `feat:` commits but no new types could be either *new-feature-on-existing* or *refactor*), present the candidates to the developer with the signals that surfaced each one and ask them to pick. Do not silently guess.

**(c) Read the abstractions (only for `new-feature-with-abstractions`).** Dispatch one or two parallel `Explore` subagents to read the new files end-to-end. Name 1–3 abstractions that the rest of the diff is wiring around. If you cannot name them in one sentence each, keep reading.

For other branch types, this sub-step is replaced by reading the *core* of the change in the type's terms:

- `new-feature-on-existing` → read the new behavior end-to-end through the affected module
- `bug-fix` → read the defect (the pre-fix code) and the fix together; understand what alternative fixes were on the table
- `refactor` → read both shapes (before and after) so you can describe the delta
- `performance` → read the hot path before and after; locate the benchmark or measurement
- `config / infra` → read every changed config file; identify the blast radius

### Phase 2 — Draft the tour (prose + diagrams together, by section)

**REQUIRED BACKGROUND:** `references/structure.md` (skeletons per branch type) and `references/diagrams.md` (Mermaid rules) — read before drafting.

Open `.agents/branch-tour.md`. Write it section by section in the order below. Render diagrams as you write the section that owns them — do not batch diagrams to the end; you will rewrite the prose around a rendered diagram, not the other way around.

**The universal section order** (present regardless of branch type):

1. **Title + 1-paragraph opener.**
2. **`## Business value`** — what changes for the user or the system. Always first after the opener.
3. **`## Scope`** — what's in this branch, what's deliberately out. Names the boundary so reviewers don't read it as broader than it is.
4. **Type-specific body** — the per-type skeleton from `references/structure.md`. Section headers depend on the branch type.
5. **Architecture diagram** (when it earns its place) — inline ` ```mermaid ` fence; see `references/diagrams.md`.
6. **`## Testing strategy`** — what's covered, what isn't, how regression is prevented. Tied to the type-specific body.
7. **`## Risks & rollout`** — breaking changes, migration steps, performance risk, observability changes. Skip cleanly only if all four are genuinely zero.
8. **`## Open questions`** — deliberate trade-offs taken, unresolved bits, decisions deferred. Capturing them in the doc pre-empts Plannotator rounds.
9. **`## Upstream artifacts`** — when the branch was built via `gzship`, link the PRD / design / plan instead of restating. *Present only if the docs exist* (check `docs/features/<slug>/` for `prd.md`, `design.md`, `plan.md`). When they exist, the *Business value* section links to `prd.md` rather than re-stating, the architecture diagram references `design.md`, and the *Scope* section quotes `plan.md`'s slice list.
10. **`## End-to-end test`** — one link to the integration spec or e2e suite that proves the change composes. A pointer, not a retelling.

**Where it's used** — when the type-specific body has a wire-up section (typical for `new-feature-with-abstractions`, `new-feature-on-existing`, and some `refactor` branches), use **`## Where it's used`** as the header. Structure it by area (subsystem, module, layer) — one subsection per area. Inside each: numbered steps with `file:line` refs and short code excerpts.

**Diagrams** (only the ones that earn their place):

- **Architecture overview** — `flowchart LR` showing the abstractions / changed shape and their relationships. Goes right after the type-specific body's main section.
- **Primary flow** — `sequenceDiagram` of the happy path / fix path / new code path. Goes inside the section it explains.
- **Lifecycle / write-path** — `sequenceDiagram` showing persistence / invalidation / regression-prevention. Only if the branch touches one.

Each diagram is rendered and visually verified (both light and dark themes) **before** being pasted as an inline fence — see `references/diagrams.md`.

Ordering rules — these are not optional:

- **Business value comes first.** Before any code or architecture.
- **More abstract abstractions come first** within the abstractions section, when one exists (Law 2 in `references/structure.md`).
- **Do NOT include a "Suggested reading order" footer.** The document IS the reading order.
- **Use names from the codebase.** Grep `HEAD` before settling on a name; honor renames.
- **Trim noisy identifiers in prose.** Keep the literal identifier inside code blocks; use the short form in narrative.

### Phase 3 — Iterate via Plannotator

Run:

```bash
plannotator annotate .agents/branch-tour.md
```

This opens the markdown in Plannotator's annotation UI. Wait for the session to close; read the annotations it returns. Address **every** annotation. Re-launch `plannotator annotate` after each substantive revision until the developer returns no more feedback.

Common annotations and how to handle them:

- *"This section reads like a changelog"* → rewrite as narrative; lead with the change, not the file.
- *"This isn't a feature, why is there a 'new abstractions' section?"* → wrong branch type. Re-classify at Phase 1 and pick the matching skeleton from `references/structure.md`.
- *"Why is this called X? Better name?"* → grep the codebase; if the code uses a different term, switch to it.
- *"Diagrams aren't rendering"* → confirm you used inline fences, not `<img>` paths; verify Plannotator is rendering mermaid; check rect/stroke colors against the current theme.
- *"Text is covered / unreadable in dark mode"* → see `references/diagrams.md` theme-safe color rules; re-render and re-verify.
- *"Order is wrong"* → re-order. The two ordering rules above are the most common offenders.
- *"What's the rollback plan?"* → config / infra change with no *Risks & rollout*. Add it.

### Phase 4 — Hand off

The tour is the deliverable. Do not auto-commit it; do not open a PR with it. **`.agents/` is gitignored** (set up at Phase 0), so the file lives only in the developer's workspace. The developer decides where it goes: PR description, design doc, internal wiki, Slack paste, or just-for-reading locally.

Print the final path and a one-line summary of what changed since the last review iteration, then stop.

If the developer asks for a follow-up edit later, treat it as a single Plannotator round: re-open the file, address the asks, re-render any diagrams whose source changed, hand back the updated file.

## What NOT to do

- Do not write a "Suggested reading order" appendix. The body IS the reading order. (Common smell: an LLM-authored tour reaches for this footer because it doesn't trust its own structure. Fix the structure instead.)
- Do not paste a diagram you have not rendered and visually inspected.
- Do not use light pastel rect colors in sequence diagrams. They are unreadable in dark theme; you will be told to redo the diagram.
- Do not reference SVG files via `<img src="diagrams/...">` or `![](diagrams/...)`. Relative path resolution in Plannotator and many markdown viewers does not work the way you'd hope.
- Do not auto-commit the tour. The tour is a deliverable for the developer to place; `.agents/` is gitignored on purpose.
- Do not invent names. If the codebase calls it `validation`, your tour calls it `validation`, even if an earlier commit called it `receipt`.
- Do not force a `new-feature-with-abstractions` skeleton onto a fix / refactor / config / perf branch. Detect the type at Phase 1 and use the right skeleton.

## Tradeoffs and design choices

- **Mermaid over D2.** Mermaid renders inline in nearly every markdown viewer that matters (GitHub, Plannotator, VS Code preview). D2 produces nicer-looking standalone SVGs but requires the viewer to either render D2 natively (rare) or be served the SVG via a reachable path (fragile in Plannotator). The tradeoff is rendering quality vs. portability; this skill picks portability.
- **Mid-tone colors over per-theme variants.** Maintaining two source files (`*.light.mmd` + `*.dark.mmd`) and swapping via `<picture>` gives prettier renders per theme, but `<picture>` doesn't reach inline mermaid fences. Mid-tone is one source, two readable renders.
- **Story over completeness.** A short tour that captures the architectural arc is more valuable than a long tour that enumerates every file. Aim for ~300–600 lines; if your tour is longer, you are probably listing instead of explaining.
- **Branch-type-aware skeleton over universal skeleton.** Not every branch is a feature with new abstractions. Half of real PRs are fixes, refactors, perf tunes, or config changes. Forcing a one-skeleton-fits-all template makes those tours awkward and the reviewer asks "why is there a 'new abstractions' section in a refactor?". The Phase 1 type-detect + per-type skeleton handles that cleanly.
- **Working scratch in `.agents/`, gitignored.** Tour markdowns and diagram intermediates are deliverables the developer hands off through other channels (PR description, design doc, wiki). The repo's git history is for *code*, not for the working notes the agent used to produce it. The single `.gitignore` change in Phase 0 keeps the working dir local-only.

## Compatibility notes

- Plannotator's annotation UI is the developer-review gate. There is no alternative; if Plannotator is not installed, this skill aborts in Phase 0.
- The `bunx`-based mermaid pipeline assumes `bun` is on PATH. If only `npm` is available, swap `bunx` → `npx -y` in all render commands; everything else is identical.
- The visual-verify step relies on macOS `qlmanage`. On Linux, substitute `rsvg-convert -h 1200 <name>.svg -o <name>.png` or `inkscape --export-type=png --export-width=1600 <name>.svg`. The rest of the pipeline is platform-neutral.
