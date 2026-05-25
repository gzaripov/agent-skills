# Writing the PRD — `gzship` Phase 2 reference

## Purpose

Phase 2 produces `docs/features/<slug>/prd.md`: the product specification for the
feature, synthesized from the conversation and the Phase 1 survey — not from a
fresh round of questions. The PRD names the problem, the solution, the user
stories, the load-bearing decisions, and the **domain terms** the rest of the
feature work will speak in.

`prd.md` is reviewed at the **Phase 3 gate** together with `scenarios.md`: a
cross-model navigator — run through the `critique-loop` skill (e.g. Codex) —
reviews both adversarially, and the developer must also approve. Write the PRD
to survive that joint review.

This file has two parts: **Rules** every PRD must obey, and **Recommendations**
applied with judgment. Follow the rules always.

## Rules — always

A PRD that breaks one of these is defective; fix it before the gate.

1. **Synthesize from context — do not re-interview.** Phase 1 already gathered
   what the feature is and how it lands. The PRD writes that down; it does not
   restart discovery. If a load-bearing fact is missing, escalate it as an open
   question, not as a new interview round.
2. **Every user story is verified by at least one scenario.** Tag each story
   with the scenarios from `scenarios.md` that prove it. A story with no scenario
   backing is either out of scope or the scenario set is incomplete — pick one
   and resolve it before the gate.
3. **No file paths and no code snippets** — those rot. Talk about modules,
   interfaces, and contracts in prose. *Exception:* a decision-encoding snippet
   produced by a throwaway prototype (a state-machine shape, a reducer signature,
   a schema), trimmed to its decision-rich parts and labeled as such.
4. **The Domain Terms section is the canonical vocabulary for this feature.**
   Every term used in scenarios, design, ADRs, code, and tests must appear here
   or in a prior PRD's Domain Terms section. The PRD is where new terms are
   introduced.
5. **Check prior PRDs before naming a new term.** Run
   `grep -l "^## Domain Terms" docs/features/*/prd.md` and read the matches. If
   a concept already has a name in a prior PRD, reuse it. If you are
   deliberately redefining a prior term, call that out in the **Aliased Terms**
   section — never silently overload it.
6. **Out of Scope is explicit, not implicit.** Name the reasonable things this
   PRD is **not** doing. A "no" with a reason is more useful at the gate than
   silence.
7. **No implementation walk-through.** The PRD captures *decisions*, not the
   order in which the code will be written. Implementation sequencing belongs
   in the design's stage breakdown, not here.

## Recommendations — apply with judgment

- **Lead with the problem from the user's perspective**, then the solution. A
  PRD that opens with the solution invites a reviewer to challenge the wrong
  thing.
- **User stories use the canonical `As an <actor>, I want <feature>, so that
  <benefit>` shape**, with the benefit as concrete as the actor.
- **One Implementation Decision per bullet** — the modules it touches, the
  interface change, the API contract, the schema choice. Keep each bullet load-
  bearing; if it can be deleted without losing meaning, delete it.
- **Testing Decisions name the kind of test, not the test framework.** What
  makes a *good* test for this feature; which modules will be tested; the closest
  prior-art tests in the repo to follow.
- **Define a term in one or two sentences.** Definitions that need a paragraph
  almost always hide a second concept — split them.
- **Be opinionated about vocabulary.** When several words mean the same thing,
  pick one and list the rest as `_Avoid_:` aliases.
- **Skip optional sections that have nothing to say.** A "Further Notes" with
  one bullet of filler is worse than no section.

## PRD template

```md
# <feature-name> — PRD

## Problem Statement

The problem from the user's perspective. One or two paragraphs.

## Solution

The solution from the user's perspective. One or two paragraphs. What the user
can now do that they could not before.

## Domain Terms

**Order**:
A confirmed customer purchase request that has entered fulfilment.
_Avoid_: Purchase, transaction.

**Invoice**:
A request for payment sent to a customer after delivery.
_Avoid_: Bill, payment request.

(Define every domain term the feature uses. Tight: one or two sentences.
Opinionated: list rejected aliases under `_Avoid_:`. Domain-only — skip general
programming concepts.)

## Aliased Terms

(Omit this section if no prior PRD term is being redefined.)

- This PRD uses **Subscription** to mean what `docs/features/2026-04-billing-v1/prd.md`
  called **Plan**. The latter is deprecated going forward.

## User Stories

1. As a <actor>, I want <feature>, so that <benefit>. *(verified by Scenarios 1, 3)*
2. As a <actor>, I want <feature>, so that <benefit>. *(verified by Scenario 5)*
3. …

(Long, numbered, exhaustive. Each story tagged with the scenarios in
`scenarios.md` that verify it.)

## Implementation Decisions

- Modules to build or modify, with the interface shape (signature, invariants,
  ordering, error modes) at each boundary.
- Schema changes.
- API contracts.
- Architectural choices made during Phase 1 discovery.

(No file paths. No code snippets except a prototype-derived
decision-encoding snippet, trimmed to decision-rich parts.)

## Testing Decisions

- What makes a good test for this feature (typically: integration-style,
  through the public interface, asserts observable behavior).
- Which modules will be tested.
- Prior art in the repo for tests of this shape.

## Out of Scope

- Explicit nos with one-line reasons. The "no" with a reason stops the next
  reviewer from re-suggesting it.

## Open Questions

(Omit if none.)

- Questions still unresolved that the developer must answer at the gate.
```

## Domain Terms — format detail

Borrowed from mattpocock/skills' `CONTEXT.md` style, scoped per-PRD:

- **Bold the term**, followed by a colon and a newline.
- **One or two sentences** describing what the term IS. Not what it does. Not
  how it is implemented.
- **`_Avoid_:`** a comma-separated list of aliases the team has rejected. This
  is what makes the vocabulary load-bearing — a reviewer can challenge any drift
  back into an avoided alias.
- **No general programming concepts.** Timeouts, error types, utility patterns
  do not belong even if the feature uses them. Only terms unique to this
  product's domain.
- **Show relationships where obvious.** A short paragraph or a dashed list
  describing how the terms relate ("An **Order** can produce one or more
  **Invoices**") is fine.

When the PRD's Domain Terms section grows past ~8 entries, group them under
subheadings (`### Orders`, `### Payments`) rather than letting it sprawl. If
multiple features in the same area keep re-declaring the same group of terms,
that is a signal to lift them — but only if the duplication is real, not
imagined.

## Aliased Terms — format detail

Only present when a prior PRD's term is being deliberately redefined or
deprecated. Format:

```
- This PRD uses **<NewTerm>** to mean what `docs/features/<prior-slug>/prd.md`
  called **<OldTerm>**. <Old or new term is deprecated going forward.>
```

A silent redefinition is worse than a deliberate one. If you are aliasing a
term, you must explain why in this section, not in prose elsewhere.

## Anti-patterns

| Smell | Why it hurts | Fix |
|---|---|---|
| Re-interviewing in Phase 2 | Phase 1 already gathered the facts; restarting discovery means the PRD is replacing scenarios, not synthesizing them | Synthesize from context; surface gaps as Open Questions |
| User stories without scenario backing | A story no scenario verifies cannot be implemented or accepted | Add the scenario or drop the story |
| File paths and code snippets in the PRD | They go stale fast; tie the PRD to a specific implementation it should not constrain | Talk about modules and contracts in prose; use snippets only for prototype-derived decision shapes |
| Implementation walk-through dressed as decisions | A step-by-step plan is a stage breakdown — that lives in the design | Capture decisions only; sequencing belongs in `design.md` |
| Domain Terms missing or hidden in prose | The rest of the feature speaks a different language than the PRD | Define every domain term in the Domain Terms section |
| Silent term overload | Two PRDs use the same word for different concepts and a future reader cannot tell | List the change in Aliased Terms with the prior PRD's path |
| Fuzzy vocabulary ("account", "user", "thing") | The reviewer cannot tell what the feature actually does | Pick one canonical term per concept; put the rest in `_Avoid_:` |
| Solution-first opening | Hides the problem and frames the review as "is this solution good?" instead of "are we solving the right thing?" | Lead with the problem; the solution paragraph comes second |
| Empty optional sections ("Further Notes: TBD") | Adds noise; reviewer cannot tell if work is missing or just unneeded | Delete the section — its absence is itself a signal |

## Sources

- mattpocock/skills' `to-prd` — Problem · Solution · User Stories · Implementation
  Decisions · Testing Decisions · Out of Scope shape; the "no file paths or code
  snippets" rule.
- mattpocock/skills' `grill-with-docs` and `CONTEXT.md` format — the ubiquitous-
  language glossary discipline, adapted here as per-PRD Domain Terms.
- *Domain-Driven Design* — Eric Evans: ubiquitous language; one canonical term
  per concept; the cost of vocabulary drift.
- *Inspired* — Marty Cagan: PRD as a synthesis of discovery, not a substitute
  for it; user-perspective problem statements.
