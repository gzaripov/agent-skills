# Writing BDD scenarios — `gzship` Phase 1 reference

## Purpose

Phase 1 produces `docs/features/<slug>/scenarios.md`: the behavioral contract for the feature. This file is the standard that artifact is held to. Use it while synthesizing the `Explore` survey and writing scenarios — every scenario should be declarative, expressed in the domain's language, and pin exactly one observable behavior. Scenarios describe *what the product does for whom*, never *how the code achieves it*.

## Discover before you write

Scenarios are *discovered*, not invented at a keyboard.

- **Three Amigos.** Good scenarios come from three perspectives — business (what problem), development (what is buildable), testing (what could break). When working solo, deliberately wear all three hats: state the business value, sanity-check feasibility, and hunt for the edge case that breaks it.
- **Example Mapping.** Break the feature into *rules*, then make each rule concrete with *examples*. Every unanswered example is a *question* — surface it rather than guessing. One scenario per example; one example per rule until a rule needs several.
- **Survey first.** Phase 1 dispatches `Explore` subagents for a reason: scenarios that ignore the existing system get rejected at the gate. Know what already exists, what the feature touches, and what it conflicts with before writing a single `Given`.

`scenarios.md` must end with a **"How it lands in the product"** section synthesized from that survey:

- **What already exists** — current behavior, modules, and patterns in this area.
- **What the feature touches** — the modules, flows, and seams it extends or modifies.
- **What it conflicts with** — existing behavior it changes or contradicts, and open questions for the developer.

## Scenario structure

Use Gherkin: a `Feature` with a short narrative, then `Scenario` blocks of `Given` (context) / `When` (the event) / `Then` (the observable outcome).

- **One observable behavior per scenario.** If the title needs "and", split it.
- **`Given` sets state, `When` is the single trigger, `Then` is the visible result.** Extra `And` is fine for setup or compound outcomes; a second `When` means a second scenario.
- **`Scenario Outline` + `Examples`** for the same behavior across an input table — one outline, many rows, no copy-paste.

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

## Declarative, not imperative

This is the single most important habit. An *imperative* scenario scripts the UI — clicks, field names, buttons. It is brittle (a redesign breaks it), it hides the intent under mechanics, and it cannot be read by the business. A *declarative* scenario states the behavior and lets the implementation choose the mechanics.

**Imperative — UI-coupled, avoid:**

```gherkin
Scenario: Apply a discount code
  Given I open the "/cart" page
  When I type "SAVE10" into the field with id "promo-input"
  And I click the "Apply" button
  And I wait for the "#total" element to update
  Then the "#total" element shows "$45.00"
```

**Declarative — business behavior, prefer:**

```gherkin
Scenario: A valid discount code reduces the order total
  Given a cart totalling 50 USD
  When the shopper applies the discount code "SAVE10"
  Then the order total is reduced by 10 percent
```

The second version survives a UI rewrite, reads as a business rule, and states *why* the number changed instead of asserting a magic string.

## Ubiquitous language

Write scenarios in the vocabulary the domain experts use — the same terms that appear in the design and the code. If the business says "shopper", "cart", and "discount code", the scenario says exactly that. Keep implementation terms (`POST /api/v2/orders`, `OrderRepository`, table names, HTTP status codes, CSS selectors) out of scenarios entirely — they belong in the design and the code, not the behavioral contract. A consistent shared vocabulary is what lets the scenario double as the spec.

## Anti-patterns

| Anti-pattern | Why it hurts | Fix |
|---|---|---|
| Imperative / UI-coupled steps (clicks, field IDs, selectors) | Breaks on any UI change; hides intent behind mechanics; unreadable by the business | State the behavior; let the implementation pick the mechanics |
| Multiple behaviors in one scenario | A failure does not localize; the title needs "and"; the scenario can't be reasoned about | Split into one scenario per observable behavior |
| Incidental detail | Noise (exact timestamps, unrelated fields) obscures what actually drives the outcome | Keep only the data the `Then` depends on; push the rest to background or defaults |
| Conjunction steps ("And X and Y…") | One step does two things, so a failure is ambiguous and the step can't be reused | One action or fact per step; chain separate `And` lines |
| Asserting implementation, not behavior | Couples the spec to internals (DB rows, status codes, log lines); refactors fail it falsely | Assert the outcome a user or stakeholder can observe |

## Sources

- *BDD in Action* — John Ferguson Smart: declarative scenarios, ubiquitous language, outside-in flow.
- *Specification by Example* — Gojko Adzic: scenarios as living, executable specifications.
- *Discovery: Explore behaviour using examples* — Seb Rose & Gáspár Nagy: Three Amigos, Example Mapping.
- *The Cucumber Book* — Matt Wynne & Aslak Hellesøy: Gherkin structure, scenario outlines, anti-patterns.
