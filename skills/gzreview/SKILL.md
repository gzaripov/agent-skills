---
name: gzreview
description: Use when producing a teammate-facing walk-through of an existing feature branch — a story-first code tour that opens with business value, surfaces the 1–3 load-bearing abstractions the branch introduces, and shows how they're injected into the existing system. Renders inline Mermaid diagrams (theme-safe mid-tone colors) and iterates via `plannotator annotate`. Triggers — "tour this branch", "code-tour the feature", "write a walk-through of the diff", "explain this PR to the team", "review my branch", "produce a reviewer-facing summary of these changes".
license: MIT
compatibility: Claude Code only — uses subagents (the Agent tool) for parallel branch exploration and task tracking. Requires the `plannotator` CLI installed (`plannotator --help`) for the iteration loop. Mermaid CLI is invoked on-demand via `bunx @mermaid-js/mermaid-cli` to verify rendered diagrams (no install needed). Run from inside a git repository, on a feature branch with commits ahead of `main`/`master`.
allowed-tools: Bash(git log *) Bash(git diff *) Bash(git status *) Bash(git rev-parse *) Bash(git branch --show-current) Bash(grep *) Bash(rg *) Bash(find *) Bash(mkdir -p .agents/*) Bash(bunx @mermaid-js/mermaid-cli*) Bash(qlmanage *) Bash(plannotator --help) Bash(plannotator annotate *) Agent
---

Produce a teammate-facing code tour of an existing feature branch and iterate it via Plannotator. The output is one markdown document that a colleague can read cold and come away understanding *why* the branch exists, *what new abstractions* it introduces, and *where they plug in*. Diagrams are inline Mermaid fences — no external SVGs to break path resolution.

## Prerequisites

- The working directory is a git repository with commits ahead of `main`/`master`.
- The `plannotator` CLI is installed and runnable (`plannotator --help`) — it runs the developer-review iteration loop.
- Network access for `bunx @mermaid-js/mermaid-cli` (one-time download per shell).
- macOS for the visual-verification step (`qlmanage` converts SVG → PNG so the rendered diagram can be inspected; on Linux, substitute `rsvg-convert` or `inkscape`).

## Core principle

**A tour is not an inventory.** A list of "Change 1, Change 2, Change 3…" is what the diff already shows. A tour is the narrative a developer would speak to a teammate in front of the diff: *"Here's why we did this, here's the new abstraction we introduced, here's where it plugs in."*

If your draft reads like a changelog, it's wrong. Rewrite it as a story.

## Output

One markdown file at `.agents/branch-tour.md` (or a path the user requests). The skill creates the file, iterates it, and leaves it as the deliverable. Diagrams live in inline ` ```mermaid ` fences inside the same file — no separate SVG / PNG artifacts in the deliverable path.

## The 7 phases

### Phase 0 — Frame & track

Confirm the working directory is a git repo on a feature branch (not `main`/`master`). Derive `<base-branch>` (default `main`; fall back to `master`). Create `.agents/` if it doesn't exist. Open a Task/Todo list with one entry per phase — the developer watches this list.

### Phase 1 — Inventory the diff

Run `git log <base>..HEAD --oneline` and `git diff <base>..HEAD --stat`. From the file list, identify:

1. **New files** — these are usually where the new abstractions live.
2. **Files with large positive deltas** — likely the injection sites where the new abstractions get wired into existing code.
3. **Test files / specs** — confirm what behavior the branch claims to add.

Read the commit messages in chronological order: they often telegraph the architectural arc (`feat:` for the abstractions, `fix:` for the wire-up, `test:` for the proof).

### Phase 2 — Identify the load-bearing abstractions

**REQUIRED BACKGROUND:** `references/structure.md` — read before drafting.

Dispatch one or two parallel `Explore` subagents to read the new files end-to-end. Your goal is to name **1–3 abstractions** that the rest of the diff is wiring around. An abstraction is something like:

- A new type / schema (e.g. a discriminated union, a Zod schema, a typed record).
- A new primitive / interface (e.g. `CompletionGuard`, `Strategy`, `Adapter`).
- A new predicate or single-source-of-truth function (e.g. `checkXReadiness(...)`).
- A new contract on an existing surface (e.g. an optional method on a `Store` interface).

If you cannot name the abstractions in one sentence each, you do not understand the branch yet. Keep reading.

### Phase 3 — Draft the tour (story-first structure)

Open `.agents/branch-tour.md`. Use this skeleton — every section is mandatory and ordered:

```
# Branch tour: <branch-name>

One-paragraph opener. File / line / commit count, single sentence about the user-visible problem.

## Business value

Why this branch exists in product terms. What was breaking, who was affected, what changes for the user. Not architecture yet.

## The N new abstractions

The whole branch is built around these. Everything else is wiring.

### Abstraction 1 — `<MostAbstractThing>`

[Sentence-level description. Then file:line reference to the declaration, then a 5-15 line code excerpt that shows the abstraction's shape.]

### Abstraction 2 — `<NextThing>`

[Same shape as Abstraction 1.]

## How the abstractions connect — at a glance

[Architecture diagram: inline ```mermaid flowchart, single source of truth.]

## Injection site 1 — Into <subsystem A>

How Abstraction 1 + 2 are wired into one part of the existing system. Code excerpts for each step.

## Injection site 2 — Into <subsystem B>

Same shape.

## Injection site 3 — Into <subsystem C>

Same shape.

## <Cross-cutting concern, if any>

E.g. the write-path / lifecycle / atomic invariants that keep the abstractions honest.

## Proof it composes

The end-to-end test / integration spec that exercises all of the above together.
```

**Ordering rules** — these are not optional:

- **Business value comes first.** Before any code or architecture. Frame the user-visible problem.
- **More abstract abstractions come first within the abstractions section.** If you have a generic primitive (e.g. `CompletionGuard`) and a domain payload (e.g. `TbxValidation`), the primitive leads.
- **Do NOT include a "Suggested reading order" footer.** The document IS the reading order. If a reader needs a footnote telling them what to read first, the body is structured wrong — fix that instead.
- **Use names from the codebase.** If the code calls it `Validation`, do not call it `Receipt` in your tour because that's what an earlier commit called it. Honor renames. Grep the current tree before settling on a name.
- **Trim noisy identifiers in prose.** `system_subagent_report_result` in prose becomes `report_result`. Keep the literal identifier inside code blocks (so file:line still grounds), but in narrative prose, use the human-friendly short form.

### Phase 4 — Add diagrams (Mermaid, inline fences)

**REQUIRED BACKGROUND:** `references/diagrams.md` — read before rendering.

Diagrams go inline as ` ```mermaid ` fences. **Not as `<img>` tags pointing at SVG files.** Relative paths from a markdown file to a sibling SVG break in Plannotator (and many other markdown viewers); inline fences are rendered by the viewer itself with no path resolution.

Three diagrams typically serve a tour well — but only add a diagram if it earns its place. Skip any that doesn't.

1. **Architecture overview** — a `flowchart LR` showing the abstractions and their relationships. Goes right after the abstractions section.
2. **Primary flow** — a `sequenceDiagram` showing the happy path through the new code. Goes inside the injection-site that owns it.
3. **Lifecycle / write-path** — a `sequenceDiagram` showing how the abstraction is persisted / invalidated / kept honest. Goes in the cross-cutting-concern section.

**Theme-safe color rules** — these matter because Plannotator (and other viewers) may render mermaid in dark theme:

- For `rect rgb(...)` highlight bands in sequence diagrams, use **mid-tone** values: `rect rgb(180, 140, 50)` for amber/warning, `rect rgb(180, 90, 90)` for red/error. Light pastels (e.g. `rgb(254, 243, 199)`) clash with dark-theme white text. Dark fills (e.g. `rgb(120, 90, 20)`) clash with light-theme black text. Mid-tone reads in both.
- For flowchart `classDef`, set **only `stroke:` and `stroke-width:`** — leave `fill:` and `color:` unset so the theme controls them.
- **Subgraph titles must fit on one line.** If your title wraps, Mermaid's dark theme renders the second line at low opacity behind the subgraph border. Shorten the title (e.g. "Three independent enforcement points" → "Enforcement points") rather than relying on the renderer to handle wrapping.

**Visual verification is required.** Do not paste a mermaid fence you have not rendered and looked at. For each diagram:

```bash
# from .agents/diagrams/ (create the directory if it doesn't exist)
bunx @mermaid-js/mermaid-cli -i <name>.mmd -o <name>.light.svg -b transparent -t default
bunx @mermaid-js/mermaid-cli -i <name>.mmd -o <name>.dark.svg  -b transparent -t dark
qlmanage -t -s 1600 -o . <name>.light.svg
qlmanage -t -s 1600 -o . <name>.dark.svg
```

Then `Read` the resulting `.png` files and confirm both themes render legibly. If a rect is unreadable, adjust the color and re-render. Only after both themes look correct, copy the verified `.mmd` content into the inline fence in the tour markdown.

The `.agents/diagrams/` working files (`*.mmd`, `*.svg`, `*.png`) are scratch — keep them or delete them at the end; they are not part of the deliverable. The single deliverable is the tour markdown with inline fences.

### Phase 5 — Iterate via Plannotator (developer review)

Run:

```bash
plannotator annotate .agents/branch-tour.md
```

This opens the markdown in Plannotator's annotation UI. Wait for the session to close; read the annotations it returns. Address **every** annotation. Re-launch `plannotator annotate` after each substantive revision until the developer returns no more feedback.

Common annotations and how to handle them:

- *"This section reads like a changelog"* → rewrite as narrative; lead with the abstraction, not the file.
- *"Why is this called X? Better name?"* → grep the codebase; if the code uses a different term, switch to it; if no better term exists, propose one and surface to the developer.
- *"Diagrams aren't rendering"* → confirm you used inline fences, not `<img>` paths; verify Plannotator is rendering mermaid; check rect/stroke colors against the current theme.
- *"Text is covered / unreadable in dark mode"* → see Phase 4 theme-safe color rules; re-render and re-verify.
- *"Order is wrong"* → re-order. The two ordering rules in Phase 3 ("business value first", "most abstract first") are the most common offenders.

### Phase 6 — Hand off

The tour is the deliverable. Do not auto-commit it; do not open a PR with it. The developer decides where it goes (PR description, design doc, internal wiki, or `.agents/` scratch). Print the final path and a one-line summary of what changed since the last review iteration, then stop.

If the developer asks for a follow-up edit later, treat it as a single Plannotator round: re-open the file, address the asks, re-render any diagrams whose source changed, hand back the updated file.

## What NOT to do

- Do not write a "Suggested reading order" appendix. The body IS the reading order. (Common smell: an LLM-authored tour reaches for this footer because it doesn't trust its own structure. Fix the structure instead.)
- Do not paste a diagram you have not rendered and visually inspected.
- Do not use light pastel rect colors in sequence diagrams. They are unreadable in dark theme; you will be told to redo the diagram.
- Do not reference SVG files via `<img src="diagrams/...">` or `![](diagrams/...)`. Relative path resolution in Plannotator and many markdown viewers does not work the way you'd hope.
- Do not auto-commit. The tour is a deliverable for the developer to place.
- Do not invent names. If the codebase calls it `validation`, your tour calls it `validation`, even if an earlier commit called it `receipt`.

## Tradeoffs and design choices

- **Mermaid over D2.** Mermaid renders inline in nearly every markdown viewer that matters (GitHub, Plannotator, VS Code preview). D2 produces nicer-looking standalone SVGs but requires the viewer to either render D2 natively (rare) or be served the SVG via a reachable path (fragile in Plannotator). The tradeoff is rendering quality vs. portability; this skill picks portability.
- **Mid-tone colors over per-theme variants.** Maintaining two source files (`*.light.mmd` + `*.dark.mmd`) and swapping via `<picture>` gives prettier renders per theme, but `<picture>` doesn't reach inline mermaid fences. Mid-tone is one source, two readable renders.
- **Story over completeness.** A short tour that captures the architectural arc is more valuable than a long tour that enumerates every file. Aim for ~300–600 lines; if your tour is longer, you are probably listing instead of explaining.

## Compatibility notes

- Plannotator's annotation UI is the developer-review gate. There is no alternative; if Plannotator is not installed, this skill aborts in Phase 0.
- The `bunx`-based mermaid pipeline assumes `bun` is on PATH. If only `npm` is available, swap `bunx` → `npx -y` in all render commands; everything else is identical.
- The visual-verify step relies on macOS `qlmanage`. On Linux, substitute `rsvg-convert -h 1200 <name>.svg -o <name>.png` or `inkscape --export-type=png --export-width=1600 <name>.svg`. The rest of the pipeline is platform-neutral.
