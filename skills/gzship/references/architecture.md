# Surveying architecture & diagramming — `gzship` Phase 3 reference

## Purpose

Phase 3 produces `docs/features/<slug>/design.md` plus one `diagrams/architecture.d2`
holding both the **current** and **proposed** architecture. This file is the standard
that work is held to. Use it while dispatching the architecture survey, drawing the D2
diagrams, and writing the design doc — survey the system as it *is* before proposing
how it *should* change, and express both states in a single diagram.

## Survey the existing architecture first

You cannot propose a sound change to a system you have not read. Before drawing a
single box, survey the codebase as it exists and pin down:

- **Module boundaries** — what units own what responsibility, and where the seams
  between them sit.
- **Data flow** — how a request/event moves through those modules, what it carries,
  and where it is transformed or persisted.
- **Design patterns already in use** — the conventions the code already follows
  (layering, dependency injection, event bus, repository, etc.). A proposal that
  ignores the resident patterns gets rejected at the design gate.
- **Prior art for similar features** — find the closest existing feature and copy its
  shape. Consistency with what exists beats a locally clever design.

When the area you must change is **untested or unfamiliar**, use the techniques from
*Working Effectively with Legacy Code* (Michael Feathers) to understand it before
proposing changes:

- **Seams.** A *seam* is a place where you can alter behavior without editing the code
  at that spot — a function boundary, an interface, an injection point. Identifying the
  existing seams tells you where a new feature can attach without invasive surgery, and
  where you would have to *create* a seam (and therefore where risk concentrates).
- **Characterization tests.** When code has no tests and its behavior is not obvious,
  write tests that *characterize* what it currently does — pin the observed behavior,
  not the intended behavior. These tests turn an unfamiliar module into a known
  quantity and become a safety net before any change. In `gzship` they feed Phase 3
  understanding; the feature's own behavior tests are written later, in Phase 5.

Record the survey result in the design doc so the proposed change is visibly anchored
to the system that exists.

## Dispatch the survey

The survey fans out — do not read the whole codebase serially in the main context.

1. **Split the question.** Turn the survey into independent sub-questions, e.g.
   "module boundaries and ownership in area X", "the data-flow path for the relevant
   request", "design patterns and conventions used here", "the closest prior-art
   feature and how it is structured".
2. **Fan out parallel `Explore` subagents** — one per sub-question, dispatched in a
   single batch so they run concurrently. Give each a tight prompt and ask for file
   paths plus a short synthesis, not raw dumps.
3. **Synthesize.** Merge the findings into one coherent picture: current boundaries,
   current data flow, resident patterns, prior art. Reconcile conflicts between
   subagent reports by reading the specific files yourself.
4. **Feed the synthesis into both deliverables** — it becomes the *current* state of
   `architecture.d2` and the opening context of `design.md`.

## D2 diagramming

Architecture diagrams in `gzship` are written in [D2](https://d2lang.com) and live in
`docs/features/<slug>/diagrams/architecture.d2`.

### Install check and fallback

Phase 3 runs the install check before rendering:

```sh
d2 --version
```

- **If `d2` is installed** — render the `.d2` to SVG (see *Render and commit* below).
- **If `d2` is missing** — offer to install it:
  - `brew install d2` (macOS / Linux with Homebrew), or
  - the official install script: `curl -fsSL https://d2lang.com/install.sh | sh -`
- **If the developer declines to install** — fall back to **source only**: commit the
  `architecture.d2` file and embed its contents as a fenced ```d2 code block inside
  `design.md`. The diagram is still reviewable as source; only the rendered SVG is
  skipped.

### D2 syntax for architecture diagrams

D2 is a declarative diagram language. The pieces you need:

- **Shapes** — a bare identifier declares a node. `label` and `shape` set its text and
  form: `api: API Gateway`, or `db: Postgres { shape: cylinder }`.
- **Nested containers** — dotted keys or nested braces group shapes. `backend.api` and
  `backend.worker` both live inside the `backend` container; the container is drawn as
  a labeled box around its children.
- **Connections with labels** — `a -> b: label` draws a directed edge with text. Use
  `--` for undirected, `<->` for bidirectional. Connections may cross containers:
  `frontend.web -> backend.api: HTTP`.
- **The `scenarios` keyword** — `scenarios` lets one file hold a **base diagram** plus
  named **scenario blocks**. Each scenario *inherits the entire base* and then applies
  its own additions and overrides on top. A scenario that names an existing shape
  modifies it; a scenario that names a new shape adds it. The base is left untouched.
  In `gzship` the base diagram is the **CURRENT** architecture and a `proposed`
  scenario is the **PROPOSED** architecture — one file, both states, no duplication.

### Complete runnable example

A base diagram (current architecture) plus a `proposed` scenario that adds a cache and
reroutes a connection:

```d2
# architecture.d2 — base = CURRENT, scenario "proposed" = PROPOSED

direction: right

client: Web Client

backend: Backend Service {
  api: HTTP API
  orders: Order Logic
}

db: Orders DB {
  shape: cylinder
}

client -> backend.api: request
backend.api -> backend.orders: dispatch
backend.orders -> db: read / write

scenarios: {
  proposed: {
    # Add a new component — read-through cache.
    cache: Order Cache {
      shape: hexagon
    }

    # Reroute: order logic now checks the cache first.
    backend.orders -> cache: lookup
    cache -> db: miss → fetch

    # Override an existing shape from the base.
    db.label: Orders DB (read replica added)
  }
}
```

Rendering this file produces two diagrams: the base (current — client → backend → db)
and the `proposed` scenario (current plus the `cache` node and its new edges).

## Render and commit

With `d2` installed, render the file. D2 renders **every scenario** in the file; point
the output at an SVG and D2 emits one file per scenario:

```sh
d2 docs/features/<slug>/diagrams/architecture.d2 \
   docs/features/<slug>/diagrams/architecture.svg
```

D2 writes the base diagram to `architecture.svg` and each named scenario to a
per-scenario file (`architecture-proposed.svg` for the `proposed` scenario). Rename or
target the outputs so the artifact set matches the expected paths:

- `docs/features/<slug>/diagrams/architecture-current.svg` — the base diagram.
- `docs/features/<slug>/diagrams/architecture-proposed.svg` — the `proposed` scenario.

Commit **both the `.d2` source and the rendered `.svg` files** with a `docs:`
conventional commit. If `d2` was unavailable, commit the `.d2` source only and embed it
fenced in `design.md` per the fallback above.

## The design document

`docs/features/<slug>/design.md` is the Phase 3 artifact and the input to the Phase 4
review gate. It must contain:

- **Components & interfaces** — the units the feature adds or changes, and the contract
  (signature, inputs, outputs) at each boundary. Anchor each to the surveyed current
  architecture and the resident patterns.
- **Data flow** — how data moves through the proposed design: the path a request/event
  takes, what it carries, where it is transformed or persisted. Reference the embedded
  current and proposed diagrams.
- **Error handling** — failure modes, how each is detected, and the recovery or
  fallback behavior. Cover the seams the change introduces.
- **Tradeoffs considered** — the alternatives weighed and why this design was chosen;
  cost, risk, and consistency with prior art. Surface unresolved architecture/product
  tradeoffs as open questions for the developer gate.
- **Implementation stage breakdown** — the feature split into ordered, independently
  testable stages. Each stage names the behavior it delivers and is small enough for
  one double-loop TDD cycle. This breakdown drives Phase 5; if it has **5 or more
  stages**, each stage is dispatched to its own fresh subagent.

Embed the rendered current and proposed diagrams (or the fenced `.d2` source under the
fallback) so the design doc is self-contained for review.

## Sources

- *Working Effectively with Legacy Code* — Michael Feathers: seams, characterization
  tests, understanding existing/untested code before changing it.
- D2 — Terrastruct: declarative diagram language, the `scenarios` keyword for
  base-plus-override diagrams, and install/render tooling
  ([terrastruct.com](https://terrastruct.com) / [d2lang.com](https://d2lang.com)).
