# Tour structure — branch type, sections, and ordering

Reference for Phase 2 of the `gzreview` skill. Read it before drafting the tour markdown. Every rule below has been earned from at least one Plannotator review rejection.

## Branch type — pick one before drafting

Not every branch introduces new abstractions. A bug fix doesn't. A refactor doesn't. A config change doesn't. The tour skeleton **branches on the branch's intent** — pick the type at Phase 1 (with the developer's confirmation) and use the corresponding body skeleton.

| Type | Telltale signals | Body skeleton |
|---|---|---|
| **new-feature-with-abstractions** | Several new files; new type / schema / interface declarations; new test files; `feat:` commits | *The new abstractions* → *Where it's used* → *Why this shape* |
| **new-feature-on-existing** | Few new files; touches one module heavily; new tests for new behavior; `feat:` commits | *What's new* → *Where it's used* → *Why this shape* |
| **bug-fix** | Tiny diff in one or two files; new regression test or modified test; `fix:` commits; commit messages cite a defect | *The defect* → *The fix* → *Why this fix and not others* |
| **refactor** | Many files moved or renamed; test deltas show preservation, not new behavior; no new types; `refactor:` commits | *The shape before* → *The shape after* → *Why* → *Behavior preserved (test list)* |
| **performance** | Targeted changes in hot paths; benchmark or perf-test deltas; `perf:` commits | *Before / After measurements* → *What changed* → *Tradeoffs* |
| **config / infra** | `package.json` / lock files / Dockerfile / CI config / `.github/` / `tsconfig.json` changes; `chore:` / `build:` / `ci:` commits | *The change* → *Blast radius* → *Rollback plan* |

When the signals are ambiguous (e.g. `feat:` commits but no new types — could be feature-on-existing or refactor), Phase 1 surfaces the candidates to the developer and asks them to pick. Do not silently guess on an ambiguous diff.

## Universal sections — present in every tour regardless of type

These wrap the type-specific body. Order is fixed.

1. **Title + 1-paragraph opener** — branch name, scope (files / lines / commits), one sentence about the user-visible problem (or the system-visible defect, for a fix).
2. **Business value** — what changes for the user or system. Always present, always first after the opener.
3. **Scope** — what's in this branch and what's deliberately out. Names the boundary so reviewers don't read it as broader than it is.
4. **Type-specific body** — picked from the table above.
5. **Architecture diagram** — inline ` ```mermaid ` fence, single source of truth. Skip if it doesn't change a reader's understanding.
6. **Testing strategy** — what's covered, what isn't, how regression is prevented. Tied to the type-specific body (e.g. for a fix: the regression test that locks the bug down).
7. **Risks & rollout** — breaking changes, migration steps, performance risk, observability changes. Skip cleanly only if all four are genuinely zero.
8. **Open questions** — deliberate trade-offs taken, unresolved bits, decisions deferred. Better to capture them in the doc than to surface them in Plannotator round after round.
9. **Upstream artifacts** — when the branch was built via `gzship`, link the PRD / design / plan instead of restating. Present iff the docs exist.
10. **End-to-end test** — one link to the integration spec or e2e suite that proves the change composes. Not a retelling — a pointer.

If a universal section has nothing to say, omit it rather than writing "N/A". Empty sections add noise; absence is itself a signal.

## The two ordering laws

### Law 1 — Business value before architecture

Open with the user-visible problem (or, for non-feature branches, the system-visible defect / inefficiency / risk). Concrete words: "the user saw X", "the system said Y but did Z", "deploys failed because…". One short paragraph, no code, no abstractions.

Why: a teammate reading the tour cold has no idea what `CompletionGuard` is or why they should care. Anchor them in product reality first; the architecture lands much harder when it follows from a problem they can see.

Anti-pattern: opening with the type signature of the new abstraction. The reader has no context to evaluate it.

### Law 2 — More abstract abstractions come first

Inside the "The new abstractions" section (only applies to `new-feature-with-abstractions` branches), the **most generic** abstraction leads. Generic means: knows the least about your specific domain, could plausibly be reused elsewhere.

Example: a branch introduces both a generic agent-loop primitive (`CompletionGuard`) and a domain-specific record (`TbxValidation`). The generic primitive leads, because the domain record is one of many possible payloads that flow through the primitive — explaining the primitive first lets you describe the domain record as "the TBX-specific payload that flows through Abstraction 1".

Reverse order forces the reader to encounter `TbxValidation` without yet knowing what gates it. They learn the predicate before they learn the gate; the gate then feels like an afterthought.

When abstractions are genuinely peers (no clear "this is built on top of that"), order by **frequency of mention** — the one referenced most in the rest of the tour goes first.

## "Where it's used" — the universal wire-up section name

When a branch's type-specific body has a wire-up / call-sites section (typically `new-feature-with-abstractions`, `new-feature-on-existing`, and many `refactor` branches), use **`## Where it's used`** as the section header. Plain English, no framework jargon. The previous "Injection sites" name read as hacky and pretended every branch wired things into multiple subsystems; this name doesn't.

Inside the section, structure by **area** (subsystem, module, layer) — one subsection per area touched. Inside each area: numbered steps with `file:line` refs and short code excerpts.

For branch types where there isn't a meaningful wire-up section (`bug-fix`, `performance`, `config / infra`), the type's own body sections (e.g. *The fix*, *What changed*) carry that content. Don't force a `Where it's used` section where it doesn't fit.

## Type-specific body skeletons — detail

### new-feature-with-abstractions

```
## The new abstractions

The whole branch is built around these. Everything else is wiring.

### Abstraction 1 — `<MostAbstractThing>`
<1 sentence framing. `**File:** path:lines`. 5–15 line code excerpt.>

### Abstraction 2 — `<NextThing>`
<Same shape.>

## Where it's used

### Into <subsystem A>
<numbered steps with file:line refs and short code excerpts.>

### Into <subsystem B>
<Same shape.>

## Why this shape — the trade-off
<Optional. One paragraph on what was rejected and why this won.>
```

### new-feature-on-existing

No new abstractions to introduce — the branch slots into shapes the codebase already has.

```
## What's new

One paragraph naming the new behavior. Skip if the Business value section already covers it tightly.

## Where it's used

### Into <subsystem the feature lives in>
<numbered steps. file:line. excerpts.>

## Why this shape — the trade-off
<Optional.>
```

### bug-fix

```
## The defect

What was wrong, in user-observable or system-observable terms. Cite the report (issue / Sentry / customer message) if there is one.

## The fix

Where the fix lands (file:line), what the change is, and the smallest excerpt that shows the corrected behavior.

## Why this fix and not others

The alternatives the developer considered and why this one won. Stops the next reader from suggesting a fix you've already rejected.
```

### refactor

```
## The shape before

What the code looked like coming in. One excerpt, just the load-bearing piece.

## The shape after

What it looks like now. Same excerpt, post-refactor.

## Why

The friction the old shape caused, the leverage the new shape gives.

## Behavior preserved

Bulleted list of the tests that lock current behavior in. If no characterization tests were written before the refactor, name that as a risk in *Risks & rollout*.

## Where it's used  (optional)

Only when the refactor touched many call sites and the touch list matters. Otherwise skip — the shape change tells the story.
```

### performance

```
## Before / After measurements

A small table: scenario · before · after · delta · methodology (1 line each). Numbers, not adjectives.

## What changed

The diff in concrete terms — the algorithm switch, the data-structure change, the cache that was added, the allocation that was removed.

## Tradeoffs

What got worse — memory, complexity, dependency surface, code-readability. A perf branch that claims to win on every axis is hiding something.
```

### config / infra

```
## The change

What was changed in `package.json` / Dockerfile / CI / lockfile / etc. Concrete diff in 1–2 paragraphs.

## Blast radius

What this affects. Builds? Deploys? Developer machines? Runtime behavior? Production data?

## Rollback plan

The exact commands or steps to revert if this lands and breaks something. A config branch without a rollback plan is a hope.
```

## Honor the codebase's naming

Grep the current `HEAD` for the term you're about to use. If the code now calls it `validation`, you call it `validation` in the tour. If an earlier commit called it `receipt` but the rename has landed (`git log --oneline | grep -i 'rename'`), you respect the rename.

Plannotator will flag this almost every time. Save the round trip: grep first.

Where to look:
- File names (e.g. `validation.ts`, not `receipt.ts`)
- Type names (e.g. `TbxValidationSchema`, not `TbxReceiptSchema`)
- Field names on the type (e.g. `validatedTemplate`, not `receiptTemplate`)
- Function names that consume it (e.g. `checkTbxValidationReadiness`)

If three of those agree, that's the name. Use it.

## Trim noisy identifiers in narrative prose

The agent tool registry uses long, mechanically-generated names like `system_subagent_report_result`. In narrative prose, refer to it as `report_result` (the short form the developer would actually say out loud). Inside code blocks, keep the literal identifier so file:line references stay accurate.

Rationale: prose is for humans. The literal name belongs in the wire-level place (the code block); the narrative voice should be the team voice.

## Size discipline

A useful tour is roughly 300–600 lines. Longer than that, you've drifted into changelog mode — every file is getting its own subsection. Cut.

Cuts to consider:
- Drop subsections that are wiring boilerplate (e.g. "and this same change was made in 7 other toolkit files; here is a table").
- Drop diagrams that don't change the reader's understanding.
- Drop code excerpts that aren't load-bearing — if a function is called but you never reference its body again, the file:line is enough.

A tour that fits on one screen of scrolling is better than one that doesn't.

## Headline test

After drafting, read just the H2 headlines top to bottom. They should tell the story by themselves. If they read like:

> Change 1 / Change 2 / Change 3 / Change 4 / …

…that's an inventory, not a tour. Rewrite.

If they read like (new-feature-with-abstractions):

> Business value / Scope / The new abstractions / Where it's used / Testing strategy / Risks & rollout / Open questions / Upstream artifacts / End-to-end test

…that's a tour. The H2s for a `bug-fix` tour read differently (Business value / Scope / The defect / The fix / Why this fix / Testing strategy / Risks & rollout / Open questions / End-to-end test) — but they still tell the story by themselves.

## Plannotator feedback patterns

These come back over and over. Pre-empt them:

- **"This reads like a changelog."** You wrote section headers like "Change 1: foo". Rewrite as narrative.
- **"Why are we still calling this X?"** You used a term the rename eliminated. Grep and fix.
- **"Why is the subagent prefix here?"** You used a noisy identifier in prose. Trim it.
- **"The abstract thing should come first."** You ordered abstractions wrong. Apply Law 2.
- **"Business value should come first."** You opened with the architecture. Apply Law 1.
- **"Remove the reading-order footer — the doc is already in this order."** Delete it; trust your H2s.
- **"This branch isn't a feature, why are there 'new abstractions' here?"** Wrong branch type. Re-classify at Phase 1 and pick the matching skeleton.
- **"What's the rollback plan?"** Config / infra change with no *Risks & rollout*. Add it.
