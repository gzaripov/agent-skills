# Writing BDD scenarios — `gzship` Phase 1 reference

## Purpose

Phase 1 produces `docs/features/<slug>/scenarios.md`: the behavioral contract for
the feature, written before any architecture or code. A scenario is *a concrete
example of system behavior from a user's perspective* — the spec, the acceptance
criteria, and living documentation in one.

`scenarios.md` feeds straight into Phase 2 (PRD synthesis), where it is paired
with `docs/features/<slug>/prd.md`; both are reviewed together at the
**Phase 3 gate**: a cross-model navigator — run through the `critique-loop`
skill (e.g. Codex) — reviews them adversarially, and the developer must also
approve, before architecture begins. Write the scenarios to survive that joint
review.

This file has two parts: **Rules** that every scenario must obey, and
**Recommendations** applied with judgment. Follow the rules always; reach for the
recommendations when they fit.

## Before you write — gather the vocabulary

Scenarios must use the same terms as every prior feature in the repo, or the
ubiquitous language drifts and reviewers can't pattern-match against earlier
work. Before drafting a scenario:

1. **Grep prior PRDs for the canonical terms**:

   ```sh
   grep -l "^## Domain Terms" docs/features/*/prd.md
   ```

   Read each match's `## Domain Terms` section. These are the canonical names
   for concepts in this codebase — reuse them verbatim in scenarios, including
   the `_Avoid_:` aliases (write the canonical term, not the avoided one).

2. **If a concept the feature touches has no name yet,** that's fine — the
   Phase 2 PRD is where you'll introduce it. Pick a tight, opinionated name now
   and use it consistently across the scenarios; you'll formalize it in the
   PRD's Domain Terms section.

3. **If you are deliberately redefining a prior PRD's term,** note it for the
   PRD's **Aliased Terms** section (see `references/prd.md`). Silent overload of
   a vocabulary term is a defect; explicit aliasing is fine.

Terms gathered here flow downstream — into `prd.md` Domain Terms, into the
design's component names, into test names, into code identifiers. Pick well.

## Rules — always

A scenario that breaks one of these is defective; fix it before the gate.

1. **State the feature's product reason.** Open the `Feature` with *why* it exists
   — who it serves and the business value it delivers — not just what it does.
   Without the "why", the scenario set cannot be scoped or judged.
2. **One scenario, one behavior.** A scenario illustrates exactly one rule. If it
   needs a second `When`→`Then` pair, or its title needs "and", split it.
3. **Describe behavior, never mechanics.** State *what* the product does, not *how*
   a user or the code achieves it. No clicks, keystrokes, field names, CSS
   selectors, URLs, or HTTP verbs in the scenario.
4. **`Given` = state, `When` = one action, `Then` = an observable outcome.** Exactly
   one triggering `When` per scenario. `Given` establishes prior context; `Then`
   states the visible result.
5. **`Then` asserts only what a user or stakeholder can observe.** Never assert
   implementation internals — database rows, log lines, status codes, private
   state, framework calls.
6. **Use the domain's ubiquitous language.** Every noun and verb is a business term
   shared by business, development, and testing — the same words the design and
   code will use. Reuse canonical terms from prior PRDs' `## Domain Terms`
   sections (see *Before you write*); introduce new ones consistently and
   formalize them in the Phase 2 PRD.
7. **Each scenario stands alone.** It is understandable by someone who has never
   seen the feature, depends on no other scenario's leftover state, and passes in
   any order.
8. **Wrap every `Feature` in a fenced ```` ```gherkin ```` code block.** Plain
   indented Gherkin renders as collapsed prose in markdown viewers and review
   tools — the reviewer literally cannot see `Given`/`When`/`Then` structure on
   the rendered page. The fence preserves indentation and turns on Gherkin
   syntax highlighting. See *Writing the Gherkin* below.
9. **System invariants are `Rule:` blocks, not standalone `Scenario:` blocks.**
   A `Scenario:` (or `Example:`) is a *single concrete user-perspective walk-
   through* of the system. A constraint the system always satisfies — "Plan
   Mode never mutates pipeline state", "an Order cannot ship before payment" —
   is a **Rule**, not a scenario. Group such invariants under a `Rule:` block
   and put one or more `Example:` blocks inside that demonstrate the rule. A
   scenario titled like a system property ("X is forbidden", "Y is required",
   "Z survives a restart") is almost always a Rule waiting to be hoisted.
   See *Writing the Gherkin* below.

## Recommendations — apply with judgment

- **Discover before you formulate.** Run the feature through the Three Amigos
  (business / development / testing perspectives — wear all three when solo) and
  Example Mapping (rules → examples → open questions) *before* writing Gherkin.
- **`Scenario Outline` for one behavior across an input table** — same rule, varied
  data. Use *separate* scenarios when the cases exercise *different* rules.
- **`Background` only for setup that is short, genuinely common, and needed to
  understand every scenario** in the file. Otherwise inline it.
- **Keep scenarios short** — aim 3–5 steps, rarely above ~10. A long scenario
  usually hides multiple behaviors or incidental detail.
- **Prefer named personas and concrete, stable data** ("a gift card worth 50 USD")
  over abstract or incidental values.
- **Title the scenario with the rule it illustrates**, not "Test X".
- **Cover the representative cases** — the happy path plus the boundaries that
  change behavior — not every permutation.

## Writing the Gherkin

A `Feature` opens with its **product reason** — who it serves and *why* it matters
(Rule 1) — then `Scenario` blocks of `Given` / `When` / `Then`. Extra `And` lines
are fine for setup or compound outcomes; a second `When` means a second scenario.

Always wrap the whole `Feature` (product-reason paragraph and every `Scenario`)
in a single fenced ```` ```gherkin ```` block (Rule 8). One block per `Feature`
is the default; do not interleave Gherkin and prose inside one block, and do not
leave Gherkin unfenced.

### Rules vs. Scenarios (Rule 9)

Gherkin 6+ supports a `Rule:` keyword that groups one or more `Example:` (the
modern synonym for `Scenario:`) blocks under a single invariant. Use it when
the thing you would otherwise call a scenario is actually a *property of the
system*, and pair it with one or more `Example:` blocks that show the property
in action from a user's perspective:

```gherkin
Feature: Gift card checkout

  ...product reason...

  Rule: A gift card balance cannot go negative

    Example: Gift card covers the full order
      Given a shopper with a gift card worth 50 USD
      And a cart totalling 40 USD
      When the shopper pays with the gift card
      Then the order is confirmed
      And the remaining gift card balance is 10 USD

    Example: Gift card does not cover the order
      Given a shopper with a gift card worth 30 USD
      And a cart totalling 40 USD
      When the shopper pays with the gift card
      Then the shopper is asked to cover the remaining 10 USD

  Rule: Refunds restore the gift card balance

    Example: Refund credits the original gift card
      ...
```

A `Rule:` block is not required when the `Feature` is a sequence of related
user flows that don't share a common invariant — a flat list of `Scenario:`
blocks is fine in that case. Reach for `Rule:` when (a) one of your scenarios
reads like a system property rather than a walkthrough, or (b) several
scenarios are demonstrating the same constraint and the title of each starts
with the same noun phrase.

```gherkin
Feature: Gift card checkout

  Shoppers receive gift cards but today cannot spend them online — the balance
  sits unredeemed and they drop out at the payment step. Accepting a gift card
  at checkout recovers that lost sale and draws down outstanding liability.

  Scenario: Gift card covers the full order
    Given a shopper with a gift card worth 50 USD
    And a cart totalling 40 USD
    When the shopper pays with the gift card
    Then the order is confirmed
    And the remaining gift card balance is 10 USD

  Scenario: Gift card does not cover the order
    Given a shopper with a gift card worth 30 USD
    And a cart totalling 40 USD
    When the shopper pays with the gift card
    Then the shopper is asked to cover the remaining 10 USD
```

Keep each scenario as plain `Given`/`When`/`Then` text. When *one* behavior spans
many inputs, a `Scenario Outline` with an `Examples` table avoids copy-paste — but
reach for it only then (see Recommendations); plain scenarios are the default.

**Declarative beats imperative** — Rule 3 in practice. An imperative scenario
scripts the UI; it breaks on any redesign and hides the intent:

```gherkin
# Imperative — UI-coupled, avoid:
Scenario: Apply a discount code
  Given I open the "/cart" page
  When I type "SAVE10" into the field with id "promo-input"
  And I click the "Apply" button
  Then the "#total" element shows "$45.00"

# Declarative — business behavior, prefer:
Scenario: A valid discount code reduces the order total
  Given a cart totalling 50 USD
  When the shopper applies the discount code "SAVE10"
  Then the order total is reduced by 10 percent
```

## Anti-patterns

| Smell | Why it hurts | Fix |
|---|---|---|
| Imperative / UI-coupled steps (clicks, field IDs, selectors) | Breaks on any UI change; hides intent; unreadable by the business | State the behavior; push mechanics into step definitions |
| Multiple behaviors in one scenario | A failure doesn't localize; the title needs "and" | One scenario per observable behavior |
| Incidental detail | Noise (timestamps, unrelated fields) obscures what drives the outcome | Keep only data the `Then` depends on |
| Conjunction steps ("Given X and Y and Z") | One step does several things; failures are ambiguous | One fact or action per step; chain separate `And` lines |
| Asserting implementation, not behavior | Couples the spec to internals; refactors fail it falsely | Assert an outcome a user or stakeholder can observe |
| Scripting instead of specifying | A click-by-click walkthrough, not an example of behavior | Rewrite as *what* outcome, not *what sequence* |
| Order-dependent scenarios | Scenario B passes only if A ran first | Each scenario sets up its own state |
| "Test X" titles | The title names a mechanic, not a rule | Title the behavior being illustrated |
| Unfenced Gherkin in `scenarios.md` | Markdown renderers collapse the indentation and the reviewer sees a wall of prose, not steps | Wrap every `Feature` in a fenced ```` ```gherkin ```` block (Rule 8) |
| `Scenario:` block that reads like a system property ("X is forbidden", "Y survives a restart") | Conflates a user-perspective walkthrough with an invariant; the reviewer cannot tell whether one passing example is enough | Hoist the property to a `Rule:` block and put `Example:` blocks under it that demonstrate the rule (Rule 9) |

## "How it lands in the product"

`scenarios.md` must end with a **"How it lands in the product"** section,
synthesized from the Phase 1 survey:

- **What already exists** — current behavior, modules, and patterns in this area.
- **What the feature touches** — the modules, flows, and seams it extends.
- **What it conflicts with** — existing behavior it changes, and open questions for
  the developer. Scenarios that ignore the existing system get rejected at the gate.

## Sources

- *BDD in Action* — John Ferguson Smart: declarative scenarios, ubiquitous language.
- *Specification by Example* — Gojko Adzic: scenarios as living, executable specs.
- *Discovery* / *Formulation* — Seb Rose & Gáspár Nagy: the Discovery → Formulation
  → Automation practices, Example Mapping.
- *The Cucumber Book* — Wynne & Hellesøy; Cucumber docs *Writing better Gherkin* /
  *Anti-patterns*: Gherkin structure, declarative style, scenario smells.
- Dan North, *Introducing BDD*; Liz Keogh on acceptance criteria vs. scenarios.
