---
name: critique-loop
description: OMP-native cross-model critique loop with two entry points. Full flow — plan → persistent navigator reviews plan → user approves → implement → same navigator reviews diff. Review-only flow — the navigator adversarially reviews an existing diff. Uses the bundled read-only critique-navigator through task and hub; no external navigator CLI.
license: MIT
compatibility: Requires OMP with the critique-loop plugin installed and an authenticated OpenAI-family provider for the bundled critique-navigator. Run inside a git repository on a feature branch, not main or master.
---

Use this skill when the user wants an adversarial cross-model review in OMP. Two flows:

- **Full flow** — plan → navigator review → user approval → implementation → navigator code review → done.
- **Review-only flow** — review and fix an existing diff without a planning or implementation phase.

This is the OMP-native variant. Use the bundled `critique-navigator` task agent and `hub`; do not invoke Codex or Cursor subprocesses.

## Configuration

- **Navigator agent:** `critique-navigator`, bundled with this plugin.
- **Artifact directory:** `.critique-loop/`, local working state that must be gitignored.
- **Slug:** current feature-branch name, or a user-approved kebab-case task identifier.
- **Plan file:** `.critique-loop/<slug>-plan.md` by default. A user may choose a repository path for a committed plan.
- **Session handle:** `.critique-loop/<slug>.agent-id`.

The bundled navigator uses `openai-codex/gpt-5.5` with `xhigh` thinking and only `read`, `grep`, and `glob`. It cannot modify files or run shell commands. The driver captures diffs into artifacts for the navigator to read.

## Navigator session operations

The workflow uses two operations:

- **START-SESSION** — spawn `critique-navigator` with the `task` tool for the first review of this task.
- **RESUME-SESSION** — send the next prompt to the same agent with `hub send` and `await: true`.

**Session reuse is mandatory.** Before START-SESSION, read `.critique-loop/<slug>.agent-id` when it exists and check whether that agent is still available. If the saved agent belongs to this task and can be resumed, use RESUME-SESSION and never overwrite it. Reuse applies across plan review, re-reviews, implementation, code review, review-only follow-ups, and later invocations that return to the same work. Continuing the chat preserves prior decisions and asks, avoids repeated discovery, and may reduce duplicate token use.

### START-SESSION

Spawn one task:

- `agent`: `critique-navigator`
- `name`: `CritiqueNavigator`
- `task`: the review prompt, verbatim

Record the actual allocated agent id returned by OMP in `.critique-loop/<slug>.agent-id`. The task result is the review; write it to the requested review artifact.

### RESUME-SESSION

Read the saved agent id, then call `hub`:

- `op`: `send`
- `to`: saved agent id
- `message`: the follow-up review prompt
- `await`: `true`

Write the reply to the requested review artifact.

A send immediately after the navigator yields may hit the park-transition race. Check the roster and retry once. Treat it as a real failure only when the retry fails and the id is absent from the live roster.

Agent revival is process-scoped. If OMP restarted, the transcript remains readable through `history://<agent-id>` but the old agent may no longer be resumable. Start a replacement only then, and seed its first prompt with the prior review, driver-response, plan, and diff-artifact paths so it can recover the earlier reasoning.

## Prerequisites

- OMP discovered both this skill and the bundled `critique-navigator` agent.
- The agent's configured OpenAI-family model is authenticated.
- Current directory is a git repository.
- Current branch is not `main` or `master`. If it is, ask for a feature-branch name and slug before continuing.

Smoke-test a new installation by spawning `critique-navigator` with `reply with exactly: OK` and confirming `OK`.

## Full flow

### Phase 1: Draft the plan

#### Step 1: Scope, slug, and artifacts

Read the request and enough repository context to write a concrete plan with real paths and existing patterns. Do not implement yet.

Derive the slug from the feature branch. Create `.critique-loop/` and ensure `.critique-loop/` is listed in `.gitignore`. If adding the ignore entry changes the repository, commit only that ignore change as `chore: gitignore .critique-loop artifacts`.

#### Step 2: Write the plan

Write the configured plan file with:

- **Task** — one paragraph in the driver's words.
- **Context** — repository constraints and prior art.
- **Approach** — numbered implementation steps naming files and symbols.
- **Files to modify** — paths with one-line reasons.
- **Verification** — commands and scenarios that prove the behavior.
- **Open questions** — decisions the driver cannot make from repository evidence.

The default plan is ephemeral under `.critique-loop/`. If the user selected a repository path, create its parent directory and commit the initial plan as `docs: plan for <slug>`.

### Phase 2: Navigator reviews the plan

#### Step 3: First plan review

Apply the session-reuse invariant. START-SESSION only when this task has no resumable agent; otherwise RESUME-SESSION.

Use output `.critique-loop/<slug>-plan-review.md` and this prompt, substituting the actual plan path:

```text
You are the navigator in an XP pair-programming session. The OMP driver wrote a plan for a task.

Read the plan at `<PLAN_FILE_PATH>` and any referenced code. Be adversarial. Probe for:
- missing edge cases and error paths
- risky assumptions or unstated dependencies
- scope mismatch
- simpler alternatives
- verification that would not catch regressions

Output:
1. One-sentence overall assessment.
2. Numbered asks. For each, state what is wrong, why it matters, and how to address it, with file:line references when possible.
3. End with exactly one verdict line and nothing after it:
   VERDICT: APPROVE
   VERDICT: CHANGES_REQUESTED
   VERDICT: BLOCK

Do not write code or modify files.
```

After START-SESSION, verify `.critique-loop/<slug>.agent-id` is non-empty before proceeding.

#### Step 4: Read the verdict

Read the review artifact's last line:

- `VERDICT: APPROVE` → user-approval gate.
- `VERDICT: CHANGES_REQUESTED` → resolve asks.
- `VERDICT: BLOCK` → summarize the blocker and stop for user input.
- Missing verdict → RESUME-SESSION once asking for the same review with a valid final verdict. Stop if it fails again.

#### Step 5: User-approval gate

Navigator approval does not authorize implementation. Present:

- slug and branch
- one-sentence implementation summary
- decisions made during review
- unresolved items from the plan's **Open questions** section
- plan path
- explicit prompt: `Ready to implement? Say 'go' to proceed, or tell me what to change.`

Wait for the user.

- Approval → Phase 3.
- Minor preference edit → revise the plan and present the gate again.
- Substantive design or scope change → revise the plan and run another navigator review in the same session.
- Stop or pivot → leave artifacts intact and stop.

#### Step 6: Resolve requested plan changes

Classify each ask.

The driver resolves directly:

- missing edge cases or failure paths
- wrong paths, symbols, or ordering
- missing verification
- simpler alternatives within the stated goal
- scope trims that remain inside the task

Surface to the user:

- product decisions
- architecture tradeoffs without a clear repository-backed answer
- business rules
- information unavailable from the repository

Revise the plan after any required user answer. Write `.critique-loop/<slug>-driver-response.md` with one line per ask, user answers verbatim, and any reasoned pushback. In repository-plan mode, commit the revision as `docs: revise plan for <slug> (round N)`.

RESUME-SESSION with output `.critique-loop/<slug>-plan-review-<N+1>.md` and this prompt:

```text
I revised the plan based on your review.
Updated plan: `<PLAN_FILE_PATH>`
Driver responses: `.critique-loop/<slug>-driver-response.md`

Re-review in the same conversation. State which prior asks are resolved and which remain open. Use the same output format and end with one VERDICT line.
```

Return to Step 4.

### Phase 3: Implement the approved plan

#### Step 7: Record the baseline

After user approval, write the current `HEAD` SHA to `.critique-loop/<slug>.plan-sha`. This is the implementation diff baseline.

#### Step 8: Implement

Implement exactly the approved plan. Do not expand scope. Commit logical changes with conventional commit messages. Run the project's relevant lint, tests, and changed-path smoke scenario before the final commit.

If implementation exposes a substantive omission or architecture decision, stop and return through the plan-review path instead of silently expanding the plan.

#### Step 9: Prepare review artifacts

Write `.critique-loop/<slug>-diff-summary.md` containing:

- baseline-to-HEAD commit range
- 2–5 significant changes
- files touched
- tests added or updated
- anything deferred

Capture the complete `git diff <PLAN_SHA>..HEAD` output and write it verbatim to `.critique-loop/<slug>-implementation.diff`. Do not ask the read-only navigator to run git commands.

### Phase 4: Navigator reviews the implementation

#### Step 10: Resume for code review

RESUME-SESSION with output `.critique-loop/<slug>-code-review.md` and this prompt, substituting the plan path and SHA:

```text
The plan you approved is implemented. Continue the same review.

Plan: `<PLAN_FILE_PATH>`
Diff summary: `.critique-loop/<slug>-diff-summary.md`
Full diff: `.critique-loop/<slug>-implementation.diff`
Commit range: `<PLAN_SHA>..HEAD`

Read the diff artifact and changed files. Verify:
1. implementation matches the approved plan
2. no scope creep
3. no bugs, regressions, or missed edge cases
4. tests cover the observable behavior

Return numbered asks with file:line references where possible and end with exactly one VERDICT line. Do not modify files.
```

#### Step 11: Handle the code-review verdict

- `VERDICT: APPROVE` → report completion.
- `VERDICT: CHANGES_REQUESTED` → resolve asks.
- `VERDICT: BLOCK` → summarize and stop for user input.

For requested changes:

- fix code-level issues directly and commit them conventionally
- surface product, architecture, or scope decisions to the user
- append responses to `.critique-loop/<slug>-driver-response.md`
- refresh `.critique-loop/<slug>-implementation.diff` with the complete baseline-to-HEAD diff
- RESUME-SESSION into `.critique-loop/<slug>-code-review-<N+1>.md`

Follow-up prompt:

```text
I addressed your code-review asks.
Driver responses: `.critique-loop/<slug>-driver-response.md`
Updated full diff: `.critique-loop/<slug>-implementation.diff`
Commit range remains `<PLAN_SHA>..HEAD`.

Re-review the full diff in this same conversation. State which prior asks are resolved and which remain open. End with exactly one VERDICT line.
```

Repeat until approval or a stopping rule fires.

#### Step 12: Report approval

Report:

- navigator verdict and round count
- implementation commit range
- tests and smoke scenario run
- any user decisions
- artifact paths

The branch is ready for the user to push, open a pull request, or merge. Do not do those unless requested.

## Review-only flow

Use this when changes already exist and the user wants only an adversarial navigator review.

### R1: Setup

Choose the slug, create and gitignore `.critique-loop/`, and apply the same session-reuse rules.

### R2: Determine and capture the diff

When intent is clear, default to the branch diff against its merge base with `origin/main`. Other supported shapes:

- working-tree changes relative to `HEAD`
- an explicit `<sha1>..<sha2>` range
- three-dot merge-base semantics

Write the chosen range to `.critique-loop/<slug>.review-range`. Capture the complete selected diff and write it verbatim to `.critique-loop/<slug>-review.diff`. Refresh this artifact after every fix.

### R3: Navigator review

START-SESSION only when no resumable session exists; otherwise RESUME-SESSION. Write the result to `.critique-loop/<slug>-code-review.md`.

Prompt:

```text
You are the navigator in an OMP cross-model code review. Continue prior context when present.

Review range: `<REVIEW_RANGE>`
Full diff: `.critique-loop/<slug>-review.diff`

Read the diff artifact and changed files. Probe for:
- bugs, regressions, and missed edge cases
- inadequate tests
- security, performance, or correctness issues
- scope creep or leftover debug code
- simpler alternatives

Return numbered asks with file:line references where possible. End with exactly one VERDICT line. Do not modify files.
```

### R4: Handle the verdict

- `VERDICT: APPROVE` → summarize review rounds and user decisions; done.
- `VERDICT: BLOCK` → summarize the blocker and stop for user input.
- `VERDICT: CHANGES_REQUESTED`:
  - fix code-level asks and commit them conventionally
  - surface product, architecture, and scope asks to the user
  - append responses to `.critique-loop/<slug>-driver-response.md`
  - refresh `.critique-loop/<slug>-review.diff`
  - RESUME-SESSION into `.critique-loop/<slug>-code-review-<N+1>.md`

Follow-up prompt:

```text
I addressed your code-review asks.
Driver responses: `.critique-loop/<slug>-driver-response.md`
Updated full diff: `.critique-loop/<slug>-review.diff`

Re-review in this same conversation. State which prior asks are resolved and which remain open. End with exactly one VERDICT line.
```

Loop until approval or a stopping rule fires. Review-only has no plan or user-approval gate.

## Stopping rules

Stop and surface the exact problem when:

- the same ask returns in three rounds without convergence
- either review phase reaches five rounds without approval
- the navigator omits a verdict twice
- task spawn fails or the configured model is unavailable
- the saved agent id is missing after START-SESSION
- a resume fails after the one park-transition retry
- a product or architecture question requires user authority
- implementation verification fails in a way requiring judgment outside the approved plan

Do not switch models, spawn a replacement chat, or weaken the navigator to obtain approval. A replacement is allowed only for a different task or after an OMP process restart makes the saved agent genuinely unrecoverable; in the latter case, seed it with the prior artifacts.
