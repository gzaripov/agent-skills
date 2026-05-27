---
name: gzplan
description: Use when the developer has a small concrete task — bug fix with a known repro, refactor a helper, add a CLI flag, tighten an API response, write a regression test, update install docs — and wants a thoughtful one-shot plan before coding. NOT for full features (use `gzship`), NOT for "I don't know what I want yet" (brainstorm first), NOT for trivial fixes (just code them). Triggers — "plan this task", "let's plan before coding", "write a plan for X", "draft a plan", "/gzplan X".
license: MIT
compatibility: Claude Code only — uses subagents (the Agent tool) for optional codebase exploration. Run from inside a git repository.
allowed-tools: Bash(git log *) Bash(git diff *) Bash(git status *) Bash(git rev-parse *) Bash(git branch --show-current) Bash(grep *) Bash(rg *) Bash(find *) Bash(ls *) Bash(mkdir -p .agents/plans) Bash(git add .gitignore) Bash(git commit -m *) Agent
---

Produce one thoughtful plan markdown for a small concrete task, fast. The output is a single file at `.agents/plans/<slug>-plan.md` that a developer (or another agent) can pick up and execute. No cross-model review, no implementation, no gates.

## Prerequisites

- The working directory is a git repository.
- The task fits the niche: a concrete change at the scale of a single PR with at most a few commits. If the task is a full feature, the right tool is `gzship`. If the task is vague ("improve the auth flow"), the right move is to brainstorm first and come back here once you have a concrete task. If the task is trivial ("rename this variable"), just code it.

## Core principle

**A plan is the smallest artifact that prevents the most expensive mistake.** Twenty minutes spent writing one tight markdown file routinely catches misread requirements, missed edge cases, and "wrong tool for the job" choices that would cost hours to undo from inside a half-written diff.

`gzplan` is not a feature spec. It does not produce a PRD, a design doc, or an ADR. It produces one focused plan tied to one concrete task.

## Output

A single markdown file at `.agents/plans/<slug>-plan.md`. `.agents/` is gitignored (set up at Phase 0). The plan never lands in the repo's history — it's a working document the developer either executes themselves, hands to another agent, or runs through `critique-loop`'s review-only flow for cross-model review.

## The 3 phases

### Phase 0 — Frame & track

1. Confirm the working directory is a git repo. The current branch can be anything; `gzplan` does not require a feature branch.
2. Derive a kebab-case `<slug>` from the task — short, descriptive, file-system-safe (e.g. `skip-linter-flag`, `utc-date-render-fix`, `helper-options-refactor`).
3. Ensure `.agents/` is gitignored. On first run in a repo, add it and make a one-line commit:

   ```bash
   if ! grep -qxE '\.agents/?' .gitignore 2>/dev/null; then
     echo ".agents/" >> .gitignore
     git add .gitignore
     git commit -m "chore: gitignore .agents working directory"
   fi
   ```

   The `.gitignore` commit is the only repo-history change `gzplan` ever makes.
4. Create `.agents/plans/` if it doesn't exist; the plan will live at `.agents/plans/<slug>-plan.md`.
5. Open a Task/Todo list with one entry per phase.

**Optional grilling — only when the task is ambiguous.** If the task description leaves load-bearing decisions open (e.g. "add a way to skip the linter" — but skip when? per-command, per-repo, env var? for everyone or per-user?), interview the developer **one question at a time** until the ambiguity is resolved. For each question, give your recommended answer. Once concrete, proceed to Phase 1.

If the task is already concrete (specific behavior, specific files implied, specific verification possible), skip grilling. Don't perform ceremony you don't need.

### Phase 1 — Draft

1. **Brief exploration.** Read just enough of the codebase to ground the plan. A single `Explore` subagent (or direct reads if the area is obvious) — *not* a fan-out. The goal is to pick up the codebase's naming, patterns, prior art for similar changes, and existing tests in the affected area. This is one task, not a feature survey.
2. **Write the plan** to `.agents/plans/<slug>-plan.md` using the template below.

**Plan template:**

```md
# <task-name>

## Task

One paragraph in your own words. What we're doing and why. The "why" is the
user-visible or system-visible reason, not "because it's in the backlog."

## Context

What exists today in the relevant area — one paragraph or a short list.
Names the patterns this plan must match (file layout, naming conventions,
test framework, the closest prior-art change). A reader unfamiliar with the
area should be able to follow the *Approach* section after reading this.

## Approach

Numbered steps at the level of "edit function X in file Y to do Z." Concrete
enough that an executor can follow them without re-deriving the design.
Keep one logical change per step.

1. ...
2. ...
3. ...

## Files to touch

- `path/to/file.ts` — what changes here and why.
- `path/to/file.test.ts` — what's added.
- ...

## Verification

How we'll know it worked — concrete:
- Commands to run (`pnpm test foo`, `bun run typecheck`, etc.).
- Manual checks if any (URL to visit, CLI command to try).
- The regression test that locks the behavior in (file:line if it exists, name
  of the test to add if it doesn't).

## Risks                                     [include when scope > 1 file or touches a hot path]

What could go wrong, named explicitly:
- Adjacent area that might break (file/module + why).
- Edge cases the Approach doesn't yet cover.
- Performance / migration / data-shape concerns.

Each risk gets a one-line mitigation or an explicit *accept-as-is* with a reason.

## Open questions                            [skip if none]

Decisions the developer should weigh in on before implementation starts. Frame
each with your recommended answer:
- **<question>?** Recommend: <option>. Reason: ...
```

**Section rules:**
- *Task*, *Context*, *Approach*, *Files to touch*, *Verification* are **required**.
- *Risks* and *Open questions* are **optional** — include only when there's something real to say. Empty sections ("Risks: none identified") add noise.
- **No file paths or code snippets in *Task*.** Talk about the user-visible / system-visible change in product terms. Snippets and paths belong in *Approach* and *Files to touch*.
- **Use the codebase's naming.** Grep the current `HEAD` before settling on a name; honor any renames that have already landed. If you're proposing a new name, call it out in *Open questions*.
- **Numbered Approach steps, not narrative prose.** A plan a reader can hand to an executor needs ordered, atomic steps. If a step needs three sub-bullets to describe, split it.

### Phase 2 — Iterate & hand off

1. Show the developer the path and a 2-line summary of what the plan covers. Wait for their feedback.
2. Iterate based on feedback — no Plannotator, no `critique-loop` round, just chat. Revise the plan markdown directly.
3. Loop until the developer says "go", "good", "ship it", or equivalent. Then stop.
4. **Optional next steps**, mentioned in the hand-off message (not invoked automatically):
   - *Execute the plan yourself, or hand it to another agent.* The plan is self-contained — paste it into a fresh chat if you want a clean executor context.
   - *Cross-model review of the plan before implementing?* Run `critique-loop`'s review-only flow on `.agents/plans/<slug>-plan.md` to get a navigator (Codex or Cursor) to adversarially review the plan. Useful when the change is small but high-stakes (a fix on a hot path, a refactor near a production boundary).
   - *Want the whole feature treatment?* Use `gzship` instead — that's PRD + design + plan + slices + gates for genuine features. `gzplan` is the lighter tool for smaller tasks.

The plan file lives at `.agents/plans/<slug>-plan.md`. The developer decides what to do with it; `gzplan` doesn't touch it again after Phase 2 ends.

## What NOT to do

- **Do not turn a feature into a `gzplan`.** If the task description names multiple user stories, touches three or more modules, or hides architectural decisions ("we'll need a new auth abstraction" — that's an abstraction), it's a feature. Stop and recommend `gzship`.
- **Do not commit the plan.** `.agents/` is gitignored on purpose; the plan is a working artifact, not repo history.
- **Do not perform a full architecture survey.** `gzplan` reads just enough to pick up naming and patterns. If you need parallel `Explore` subagents fanning out across the codebase, the task is too big for `gzplan`.
- **Do not embed the design rationale here.** The plan says *what* and *how*, briefly. Long "why we chose this approach over the alternatives" reasoning belongs in an ADR or a design doc — and if you need that, the task is too big for `gzplan`.
- **Do not invent verification.** If you can't name a concrete way to verify the change (a command, a test, a manual check), the *Verification* section is incomplete — surface it in *Open questions*.
- **Do not vibe-grill.** If the task is concrete, skip the Phase 0 grilling pass. Optional grilling is for genuinely ambiguous tasks, not for ceremony.

## When `gzplan` is the wrong tool

Send the developer somewhere else when:

| If the task is… | Use this instead |
|---|---|
| A whole feature (multiple stories, multiple modules, architectural decisions) | `gzship` |
| Vague — "improve the auth flow", "make this faster" | Brainstorm first; come back once concrete |
| Trivial — one variable rename, one typo fix | Just code it |
| Reviewing changes already made | `critique-loop` review-only flow, or `gzreview` for a teammate-facing tour |
| Babysitting an open PR to merge | `babysit-pr` |

Naming the wrong tool is more useful than producing a wrong-shaped plan. Be willing to say "this is too big for `gzplan` — use `gzship`" or "this is too small for `gzplan` — just code it."

## Tradeoffs and design choices

- **Single document, no review gates.** `critique-loop`'s full flow already exists for plan + navigator review + implementation. `gzplan` deliberately stops at the plan and skips the navigator — the developer can compose with `critique-loop` afterward if they want review, but most small tasks don't need it.
- **Gitignored, not committed.** Plans for small tasks rot fast — the change ships, the plan becomes stale. Committing them adds noise. The plan is a hand-off document, not a paper trail.
- **One subagent at most, not a fan-out.** A small task doesn't need a feature survey. One `Explore` subagent is enough to pick up patterns and naming; more is overkill.
- **Grilling is optional, not mandatory.** Forcing a grilling pass on a concrete task wastes time. The agent decides per-task whether the description has load-bearing ambiguity; if not, straight to draft.
- **No PRD, no design, no ADRs.** Those are `gzship` artifacts. If the task warrants them, the task warrants `gzship`, and `gzplan` recommends that escalation instead of half-implementing it.

## Compatibility notes

- This skill produces only the plan markdown. Whether and how to execute it is the developer's call. Plans are language-agnostic, so the executor can be Claude in a fresh session, a teammate, or `critique-loop`'s implementation phase.
- `.agents/` is shared with `gzreview`. The `.gitignore` step is a no-op when `gzreview` (or any other skill that needs `.agents/` gitignored) has already added the entry.
