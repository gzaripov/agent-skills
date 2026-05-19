# Writing BDD scenarios — `gzship` Phase 1 reference

## Purpose

Phase 1 produces `docs/features/<slug>/scenarios.md`: the behavioral contract for
the feature, written before any architecture or code. A scenario is *a concrete
example of system behavior from a user's perspective* — the spec, the acceptance
criteria, and living documentation in one.

This file has two parts: **Rules** that every scenario must obey, and
**Recommendations** applied with judgment. Follow the rules always; reach for the
recommendations when they fit.

## Rules — always

A scenario that breaks one of these is defective; fix it before the gate.

1. **One scenario, one behavior.** A scenario illustrates exactly one rule. If it
   needs a second `When`→`Then` pair, or its title needs "and", split it.
2. **Describe behavior, never mechanics.** State *what* the product does, not *how*
   a user or the code achieves it. No clicks, keystrokes, field names, CSS
   selectors, URLs, or HTTP verbs in the scenario.
3. **`Given` = state, `When` = one action, `Then` = an observable outcome.** Exactly
   one triggering `When` per scenario. `Given` establishes prior context; `Then`
   states the visible result.
4. **`Then` asserts only what a user or stakeholder can observe.** Never assert
   implementation internals — database rows, log lines, status codes, private
   state, framework calls.
5. **Use the domain's ubiquitous language.** Every noun and verb is a business term
   shared by business, development, and testing — the same words the design and
   code will use.
6. **Each scenario stands alone.** It is understandable by someone who has never
   seen the feature, depends on no other scenario's leftover state, and passes in
   any order.

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

A `Feature` with a short narrative, then `Scenario` blocks of `Given` / `When` /
`Then`. Extra `And` lines are fine for setup or compound outcomes; a second `When`
means a second scenario.

```gherkin
Feature: Gift card checkout
  Shoppers can pay with a gift card balance.

  Scenario: Gift card covers the full order
    Given a shopper with a gift card worth 50 USD
    And a cart totalling 40 USD
    When the shopper pays with the gift card
    Then the order is confirmed
    And the remaining gift card balance is 10 USD

  Scenario Outline: Insufficient gift card balance
    Given a shopper with a gift card worth <balance> USD
    And a cart totalling <total> USD
    When the shopper pays with the gift card
    Then the shopper is asked to cover the remaining <shortfall> USD

    Examples:
      | balance | total | shortfall |
      | 30      | 40    | 10        |
      | 0       | 25    | 25        |
```

**Declarative beats imperative** — Rule 2 in practice. An imperative scenario
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
