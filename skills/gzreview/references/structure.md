# Tour structure — the rules and the rationale

This is the reference for Phase 3 of the `gzreview` skill. Read it before drafting the tour markdown. Every rule below has been earned from at least one Plannotator review rejection.

## The two ordering laws

### Law 1 — Business value before architecture

Open with the user-visible problem the branch closes. Concrete words: "the user saw X", "the system said Y but did Z", "deploys failed because…". One short paragraph, no code, no abstractions.

Why: a teammate reading the tour cold has no idea what `CompletionGuard` is or why they should care about a `TbxValidation` record. Anchor them in product reality first; the architecture lands much harder when it follows from a problem they can see.

Anti-pattern: opening with the type signature of the new abstraction. The reader has no context to evaluate it.

### Law 2 — More abstract abstractions come first

Inside the "N new abstractions" section, the **most generic** abstraction leads. Generic means: knows the least about your specific domain, could plausibly be reused elsewhere.

Example: a branch introduces both a generic agent-loop primitive (`CompletionGuard`) and a domain-specific record (`TbxValidation`). The generic primitive leads, because the domain record is one of many possible payloads that flow through the generic primitive — explaining the primitive first lets you describe the domain record as "the TBX-specific payload that flows through Abstraction 1".

Reverse order forces the reader to encounter `TbxValidation` without yet knowing what gates it. They learn the predicate before they learn the gate; the gate then feels like an afterthought.

When abstractions are genuinely peers (no clear "this is built on top of that"), order by **frequency of mention** — the one referenced most in the rest of the tour goes first.

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

## The skeleton

Use this exact ordering. Skipping a section is fine if the branch doesn't have content for it; reordering is not.

1. **Title + 1-paragraph opener** — branch name, scope (files / lines / commits), one sentence about the user-visible problem.
2. **Business value** — Law 1.
3. **The N new abstractions** — one subsection per abstraction, in order per Law 2. Each subsection: 1 sentence framing → `**File:** path:lines` → fenced code excerpt (5–15 lines).
4. **Architecture diagram** — inline mermaid fence, single source of truth, shows the abstractions and their relationships.
5. **Injection sites** — one section per place the abstractions plug into existing code. Inside each: numbered steps with file:line refs and code excerpts.
6. **Cross-cutting concerns** — write-path, lifecycle, invariants. Optional. Only if the branch has them.
7. **Proof it composes** — the end-to-end test or integration spec that exercises the whole thing.

What goes **at the bottom of the file** is the integration test reference. Not a "suggested reading order" footer.

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

If they read like:

> Business value / The new abstractions / How they connect / Where they plug in (3×) / The proof

…that's a tour.

## Plannotator feedback patterns

These come back over and over. Pre-empt them:

- **"This reads like a changelog."** You wrote section headers like "Change 1: foo". Rewrite as narrative.
- **"Why are we still calling this X?"** You used a term the rename eliminated. Grep and fix.
- **"Why is the subagent prefix here?"** You used a noisy identifier in prose. Trim it.
- **"The abstract thing should come first."** You ordered abstractions wrong. Apply Law 2.
- **"Business value should come first."** You opened with the architecture. Apply Law 1.
- **"Remove the reading-order footer — the doc is already in this order."** Delete it; trust your H2s.
