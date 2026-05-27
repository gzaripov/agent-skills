# Surveying architecture & writing the design — `gzship` Phase 4 reference

## Purpose

Phase 4 produces `docs/features/<slug>/design.md` plus an architecture diagram
showing both the **current** and **proposed** architecture, and (when warranted)
new entries under `docs/adr/`. The design doc is the input to the **Phase 5
gate**: a cross-model navigator — run through the `critique-loop` skill (e.g.
Codex) — reviews it adversarially, and the developer must also approve it,
before implementation begins. It must let a reviewer understand *why* this
design, not just *what* it is.

This file has two parts: **Rules** every design must obey, and **Recommendations**
applied with judgment. Follow the rules always.

## Rules — always

A design doc that breaks one of these fails the gate.

1. **Survey the existing architecture before proposing a change.** Describe the
   system as-is — module boundaries, data flow, resident patterns — before the
   to-be. Never design against a system you have not read.
2. **State Goals and Non-Goals explicitly.** Non-goals are reasonable objectives
   deliberately excluded; naming them forces a real scope decision.
3. **Every design element traces to an approved scenario.** Anything in the design
   that no scenario calls for is scope creep — cut it or justify it.
4. **State the alternatives considered — including "do nothing" — and why this one
   won.** A design with no alternatives section is not reviewable.
5. **Be tradeoff-centric.** For the chosen design, state what it *costs*, not only
   what it buys. A doc that only sells is defective.
6. **Make assumptions and failure modes explicit.** Write down dependencies, load,
   and data-shape assumptions; enumerate how the design can fail and how each
   failure is handled or knowingly accepted.
7. **Surface unresolved open questions.** List them plainly; never bury uncertainty
   in prose or silently design past it. Product/architecture questions go to the
   developer at the gate.

## Recommendations — apply with judgment

- **Size the doc to the change** — a one-page mini-doc for a small feature; long
  form only for large ones. As short as possible, as long as necessary.
- **Lead with a one-paragraph overview / TL;DR** before the detailed design.
- **Show current vs. proposed side by side** — paired diagrams make the delta
  reviewable at a glance.
- **Sketch APIs and data shapes, don't paste them** — include only the parts that
  drive a tradeoff; link full schemas.
- **For legacy or untested code, name the seams and a characterization-test plan** —
  how you will pin current behavior before changing it.
- **Record architecture-significant decisions as ADRs** — sparingly, only when
  the decision meets all three offer criteria (see *Architecture Decision
  Records*). One per decision; immutable once accepted.
- **Address cross-cutting concerns** — security, observability, data migration —
  or mark each an explicit "N/A because…".

## Survey the existing architecture first

You cannot propose a sound change to a system you have not read. Before drawing a
box, pin down: **module boundaries** (what owns what, where the seams are),
**data flow** (how a request moves, what it carries), **resident patterns** (the
conventions the code already follows), and **prior art** (the closest existing
feature — copy its shape).

**Read existing ADRs in the area before designing.** Run `ls docs/adr/ 2>/dev/null`
and skim any entries whose slug touches this feature's area. ADRs record
deliberate decisions — re-litigating them silently is how a design loses a
reviewer's trust. If a candidate design contradicts an existing ADR, surface
that in the design doc with a callout (`*contradicts ADR-NNNN — but worth
reopening because…*`) — never just route around it.

When the area is **untested or unfamiliar**, use *Working Effectively with Legacy
Code* (Michael Feathers):

- **Seams** — a place where you can change behavior without editing code there (a
  function boundary, an interface, an injection point). Existing seams show where a
  feature can attach without invasive surgery; a missing seam shows where risk
  concentrates.
- **Characterization tests** — when code has no tests and its behavior is not
  obvious, write tests that pin *what it currently does*. They turn an unfamiliar
  module into a known quantity before any change.

## Dispatch the survey

The survey fans out — do not read the whole codebase serially in the main context.

1. **Split the question** into independent sub-questions: module boundaries,
   data-flow path, resident patterns, closest prior art.
2. **Fan out parallel `Explore` subagents** — one per sub-question, in a single
   batch. Ask each for file paths plus a short synthesis, not raw dumps.
3. **Synthesize** the findings into one picture; reconcile conflicts by reading the
   specific files yourself.
4. **Feed the synthesis** into the diagram's *current* state and the opening
   context of `design.md`.

## Diagramming — draft both tools, render both, choose

`gzship` is not tied to one diagram tool. For the architecture diagram, **draft it
in both D2 and Mermaid, render each, look at the two renders, and keep whichever
reads more clearly for this feature.** Neither tool wins every time:

- **D2** — stronger auto-layout for dense graphs and nested containers; one file
  can hold both states (`layers` / `scenarios`). Needs the `d2` binary, and GitHub
  does not render `.d2` — the rendered SVG must be committed and embedded.
- **Mermaid** — GitHub and most markdown viewers render a fenced ` ```mermaid `
  block inline, so the diagram needs no committed image. Cleaner default shapes for
  simple flows; but it wraps long labels aggressively and has no current-vs-proposed
  board (use two diagrams).

Diagram sources live in `docs/features/<slug>/diagrams/`.

### D2

Install check: `d2 --version`. If missing, offer `brew install d2` or
`curl -fsSL https://d2lang.com/install.sh | sh -`.

- **Shape** — `api: API Gateway`, or `db: Postgres { shape: cylinder }`. Pick a
  fitting shape: a plain rectangle for a normal module, a cylinder only for a
  datastore.
- **Container** — dotted keys or nested braces group shapes: `backend.api`.
- **Connection** — `a -> b: label`; edges may cross containers.
- **Two states in one file** — `scenarios` blocks inherit the base and override
  (proposed builds on current); `layers` blocks are independent boards (current and
  proposed share no structure, e.g. a greenfield repo).

Render each board with `--target` — `''` for the base, `scenarios.<name>` or
`layers.<name>` for the proposed board:

```sh
D=docs/features/<slug>/diagrams
d2 --target ''                "$D/architecture.d2" "$D/architecture-current.svg"
d2 --target 'layers.proposed' "$D/architecture.d2" "$D/architecture-proposed.svg"
```

### Mermaid

Render check: `mmdc --version`, or run on demand with `npx -y
@mermaid-js/mermaid-cli`. A `flowchart` is the usual fit:

- **Node shapes** — `id["box"]`, `id(["pill / actor"])`, `id{{"hexagon"}}`,
  `id[("cylinder")]`.
- **Edge with label** — `a -- "label" --> b`.
- **Line breaks** — `<br/>` inside a label; keep each line short, Mermaid wraps
  long lines at awkward points.
- **Two states** — current and proposed are two separate ` ```mermaid ` blocks.

Render to PNG to inspect: `npx -y @mermaid-js/mermaid-cli -i "$D/architecture.mmd"
-o /tmp/arch.png -s 2`.

### Look at what you rendered — and compare

A diagram you have not seen is not done. Render **both** versions to PNG and open
them (you can read image files). Judge each as a reader would:

- no oversized or near-empty shapes; no node far larger than its content;
- no overlapping nodes or labels; no clipped or cramped text;
- few edge crossings; related nodes sit close together;
- the layout is balanced — not jammed into one corner with dead space.

Fix the source and re-render until each reads cleanly. Then **pick the tool whose
render is clearer** — that one becomes the diagram.

## Embed and commit

Embed the **chosen** diagram in `design.md` so the doc is self-contained:

- **Mermaid chosen** — paste it into `design.md` as a fenced ` ```mermaid ` block
  (current and proposed each their own block). GitHub renders it inline; no image
  artifact needed. Keep the source as `diagrams/architecture.mmd` too.
- **D2 chosen** — embed the rendered SVGs with markdown image syntax (never a raw
  `.d2`); commit the `.d2` source and both `.svg` files.

Commit with a `docs:` conventional commit. If neither tool installs, commit the
diagram source and embed it as a fenced code block so it is reviewable as text.

## The design document

`docs/features/<slug>/design.md` must contain, anchored to the survey:

- **Context & scope** — the problem and the current architecture it lands in.
- **Goals / Non-Goals** — explicit bullets (Rule 2).
- **System Analysis** *(required — see below)* — what exists today: the
  modules in the path, the key data flow as-is, current load vs. PRD Quality
  Targets, prior art the design must respect.
- **Components & interfaces** — the units added or changed, with the contract
  (signature, inputs, outputs) at each boundary.
- **Data flow** — the path a request/event takes through the proposed design.
- **Integration & Data Ownership** *(required — see below)* — the explicit
  per-call communication style (sync / async / streaming) and per-entity
  ownership (who owns it, who has projections, what consistency model).
- **Diagrams** — current and proposed, embedded (see above).
- **Alternatives considered** — incl. "do nothing", with rejection reasons (Rule 4).
- **Tradeoffs** — what the chosen design costs (Rule 5).
- **Error handling & failure modes** — how each failure is detected and handled.
- **Risks** *(required — see below)* — risk register tied to scenarios and
  Quality Targets, with mitigation or accepted-as-is for each.
- **Assumptions & open questions** — explicit (Rules 6, 7).

The implementation slice breakdown does **not** live here — it is the Phase 6
deliverable (`plan.md`). The design feeds into the plan; the plan does not
feed back.

### System Analysis (the as-is)

The design must open by describing the system as it stands today, scoped to the
area the feature touches. This is the "as-is" the course calls non-negotiable.
Include:

- **Modules in the path** — names + one-line responsibility each. Match
  `CONTEXT.md`-style language used elsewhere in the repo (or the canonical
  Domain Terms from the PRD).
- **Key flow as-is** — for the most important scenario, the path a request /
  event takes today: which module does what, where data is read/written,
  which calls are synchronous vs. asynchronous, where external systems sit.
- **Current load vs. PRD Quality Targets** — for each quality the PRD names a
  target for (latency, RPS, volume, availability), state what we know about
  today's behavior in the same dimension. Honest "unknown — need measurement"
  is better than invented numbers.
- **Prior art** — the closest existing feature/pattern this design should
  match. Copying the resident shape is cheaper than inventing one.

A design that proposes change without this as-is description is rejected at the
gate — it is designing against an unknown system.

### Integration & Data Ownership

For any cross-module call the design introduces or changes, name two things:

**(a) Integration style and why.** One of:

- **Synchronous RPC/REST** — simple, but couples caller and callee; cascading
  failure on the callee's outage; the caller's latency budget includes the
  callee's. Pick when an immediate answer is required and the callee's
  availability is acceptable to the caller's SLO.
- **Asynchronous command/event** — loose coupling, callee outage doesn't break
  the caller, but the system becomes eventually consistent and idempotency
  must be designed in. Pick when no immediate answer is required, or when
  callee availability is below caller SLO.
- **Streaming data** — the stream itself is the contract; callee maintains a
  projection of the source of truth. Pick when many consumers need the same
  state, or the callee's read pattern is hot.
- **Shared database** — almost always wrong; lists it only to explicitly
  reject it with a reason.

**(b) Data ownership per entity.** For each domain entity the feature touches:

- **Owner** — exactly one module owns this entity's source of truth.
- **Consumers** — other modules holding projections / copies, and how they
  stay in sync (event subscription, CDC, on-demand fetch).
- **Consistency model** — strong / read-your-writes / eventual / monotonic.
  Make the trade-off explicit.

A design that leaves integration style or data ownership implicit will hit
exactly those questions at production-incident time — better to make them
review-time decisions.

### Risks

A table at the end of `design.md`, one row per risk. Tie each risk to the
scenario(s) it threatens and the Quality Target(s) it hits. Mitigation is
either a concrete plan or an explicit *accept-as-is* with a reason.

| Risk | Scenario(s) it threatens | Quality hit | Mitigation |
|---|---|---|---|
| Order-service outage cascades to checkout | Scenario 3 | Availability (99.5% target) | Async event with idempotent retry; checkout degrades to "we'll confirm by email" instead of failing |
| Catalog price drift between cache and source | Scenario 5 | Read consistency | Cache TTL ≤ 60s; explicit version field; *accepting* 60s drift |
| New PII table without DPO sign-off | — | Regulatory (GDPR) | Hold until DPO approval; track as open question |

Categories to scan for: **technological** (stale stack, single point of
failure, dangerous integration), **organisational** (one team owns too much,
no clear owner), **domain** (a module knows another's logic), **data**
(no source of truth, divergent copies). Sized to the change — a small
feature may have one or two rows; a system-shape change may have ten.

## Architecture Decision Records (ADRs)

ADRs record *that* a decision was made and *why*, in one place a future reader
will look. They are written sparingly, alongside `design.md`, at Phase 4.

### When to write one

All three of these must be true:

1. **Hard to reverse** — the cost of changing your mind later is meaningful
   (database choice, API contract, integration pattern across modules, lock-in).
2. **Surprising without context** — a future reader will look at the code and
   wonder "why on earth did they do it this way?" If the rationale is obvious
   from the code, you do not need an ADR.
3. **The result of a real trade-off** — there were genuine alternatives and you
   picked one for specific reasons. "We did the obvious thing" is not an ADR.

If any of the three is missing, skip the ADR. Most design decisions do not
warrant one.

### What qualifies

- **Architectural shape.** "We're using a monorepo." "The write model is event-
  sourced, the read model is projected into Postgres."
- **Integration patterns between modules.** "Ordering and Billing communicate
  via domain events, not synchronous HTTP."
- **Technology choices that carry lock-in.** Database, message bus, auth
  provider, deployment target. Not every library — only ones that would take a
  quarter to swap out.
- **Boundary and scope decisions.** "Customer data is owned by the Customer
  module; other modules reference it by ID only." The explicit no-s are as
  valuable as the yes-s.
- **Deliberate deviations from the obvious path.** "We're using manual SQL
  instead of an ORM because X." Anything where a reasonable reader would assume
  the opposite.
- **Constraints not visible in the code.** "We can't use AWS because of
  compliance requirements." "Response times must be under 200 ms because of the
  partner API contract."
- **Rejected alternatives when the rejection is non-obvious.** If you considered
  GraphQL and picked REST for subtle reasons, record it — otherwise someone
  will suggest GraphQL again in six months.

### Format

ADRs live in `docs/adr/` and use sequential numbering: `0001-<slug>.md`,
`0002-<slug>.md`, etc.

Lazily create the `docs/adr/` directory — only when the first ADR is needed.
Scan existing entries for the highest number and increment by one.

Template:

```md
# <Short title of the decision>

> *Originating feature:* `docs/features/<slug>/`

<1–3 sentences: what's the context, what did we decide, and why.>
```

The `Originating feature:` line is **required**. ADRs outlive features by
design, but knowing which feature introduced the decision is load-bearing for
future readers — it gives them the context to understand *why* the decision was
made and which review captured it.

An ADR body can otherwise be a single paragraph. The value is in recording
*that* a decision was made and *why* — not in filling out sections.

**Optional sections** — include only when they add genuine value; most ADRs
will not need them:

- **Status** frontmatter (`proposed | accepted | deprecated | superseded by
  ADR-NNNN`) — useful when decisions are revisited.
- **Considered Options** — only when the rejected alternatives are worth
  remembering.
- **Consequences** — only when non-obvious downstream effects need to be
  called out.
- **Revisit conditions** — the triggers that should make a future reader reopen
  this decision. *"Reopen this when monthly write volume exceeds 1 M rows,"* or
  *"reopen this when the Payments team is staffed to own an idempotency layer
  themselves."* Better than a quiet *"this is final"* — decisions age, and naming
  the trigger is honest about that. Add this section when the trigger is
  actually nameable; skip it when the decision is genuinely durable.

### Bidirectional linking

Every ADR a feature produces must be discoverable from both directions:

**Forward (feature → ADR).** Add an **`## ADRs produced`** section to
`design.md` listing every new ADR this feature wrote, with the path relative
to `docs/features/<slug>/design.md` and a one-line summary of what was
decided:

```md
## ADRs produced

- [ADR-0017 — Postgres for the write model](../../adr/0017-postgres-for-write-model.md) — chose Postgres
  over DynamoDB for the balance table because of strong-read requirements at checkout.
- [ADR-0018 — Async balance projection](../../adr/0018-async-balance-projection.md) — checkout reads a
  projection rather than the write model; accepting up-to-60 s drift for read-path availability.
```

If the feature produces no ADRs, omit the section. Do not write an empty
`## ADRs produced` with the text "none" — its absence is the signal.

**Backward (ADR → feature).** The `Originating feature:` line in the ADR
template (above) carries the path back. One ADR → one originating feature.
If a later feature *amends* the decision, it writes a new ADR that supersedes
the old one (set `Status: superseded by ADR-NNNN` on the original); the
amending ADR's `Originating feature:` points to the new feature.

### Commit alongside the design

ADRs created during Phase 4 are committed together with `design.md` in the
same `docs:` conventional commit (e.g. `docs: design for <slug>`). The
forward links in `design.md` and the backlinks in each ADR header land in
the same commit and are reviewed at the **Phase 5 gate** alongside the design.

## Anti-patterns

| Smell | Why it hurts | Fix |
|---|---|---|
| Implementation manual — *how*, no *why* or tradeoffs | Not reviewable; reviewer can't judge the choice | Add design rationale and tradeoffs; if there's no ambiguity, skip the doc |
| No alternatives considered | The choice looks inevitable; reviewer can't push back | Add ≥2 alternatives incl. "do nothing" and rejection reasons |
| Vague requirements ("fast", "scalable") | Unverifiable; means nothing at the gate | Replace adjectives with measurable targets tied to a scenario |
| Unstated assumptions | Hidden risk; the design breaks when one is wrong | List every assumption; mark validated vs. unverified |
| Scope creep | The doc balloons into adjacent systems | Add Non-Goals; move tangential concerns out |
| No failure analysis | Only the happy path is designed | Enumerate failure modes and the handling for each |
| Designs against an unknown system | Proposes change without surveying current state | Document the as-is architecture first; for untested code, name seams |
| Diagram with no legend/labels | Boxes and arrows that need narration | Title, key, labeled directional edges; verify the render |
| Buried uncertainty | Open questions hidden in prose or omitted | Hoist them into an Open Questions section |
| Untraceable design | Elements with no originating scenario | Drop or justify any element that doesn't trace to an approved scenario |

## Sources

- *Working Effectively with Legacy Code* — Michael Feathers: seams, characterization
  tests.
- "Design Docs at Google" — Malte Ubl: context/scope, goals/non-goals, alternatives,
  tradeoff-centric design, cross-cutting concerns.
- Architecture Decision Records — Michael Nygard; Martin Fowler: Context / Decision
  / Consequences, immutable once accepted. Tight 1–3-sentence template and the
  hard-to-reverse / surprising / real-trade-off offer criteria are adapted from
  mattpocock/skills' `grill-with-docs` ADR-FORMAT.
- The C4 model — Simon Brown: context diagrams, self-describing notation.
- D2 ([d2lang.com](https://d2lang.com)) and Mermaid ([mermaid.js.org](https://mermaid.js.org)):
  diagram languages.
