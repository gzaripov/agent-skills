---
name: critique-loop
description: Cross-model critique loop with two entry points. Full flow — plan → navigator reviews plan → user approves → implement → navigator reviews diff. Review-only flow — navigator adversarially reviews an existing diff; no plan, no implementation. Same navigator session across rounds so it retains context. Navigator is pluggable — Codex CLI (default, gpt-5.5 at xhigh effort) or Cursor CLI (gpt-5.3-codex-xhigh). Claude resolves code-level asks itself; product/architecture questions are surfaced to the user, then the same session resumes with the answer.
license: MIT
compatibility: Requires either Codex CLI (`codex`, authenticated with `codex login`) or Cursor CLI (`cursor-agent`, authenticated with `cursor-agent login`). Run from inside a git repository, on a feature branch (not `main`/`master`).
allowed-tools: Bash(codex exec *) Bash(codex exec resume *) Bash(cursor-agent *) Bash(git add *) Bash(git commit *) Bash(git status *) Bash(git diff *) Bash(git log *) Bash(git rev-parse *) Bash(git branch --show-current) Bash(mkdir -p .critique-loop) Bash(cat .critique-loop/*) Bash(grep -oE *) Bash(tee .critique-loop/*)
---

Use this skill when the user wants an adversarial cross-model review. Two flows:

- **Full flow** — plan → navigator reviews → (fix or ask user) → **user approves the plan** → implement → navigator reviews the diff → (fix or ask user) → done. All four phases share a single navigator session (tracked by UUID in `.critique-loop/<slug>.session-id`, resumed on every subsequent call) so the navigator remembers the plan discussion when it reviews the code. Trigger phrases: "critique-loop on <task>", "pair with Codex/Cursor on this", "plan and have Codex review", "do this with cross-model review".
- **Review-only flow** — skip planning and implementation; run just the navigator code-review loop against an existing diff. Trigger phrases: "review my changes with Codex/Cursor", "critique-review this branch", "have the navigator look at my diff". See the **Review-only flow** section at the end of this document.

## Configuration

Defaults live here — edit to override. Everything below reads from these.

- **Navigator CLI:** `codex` (options: `codex` | `cursor`)
- **Artifact directory:** `.critique-loop/` — **local working state only, gitignored**. Not committed; not part of the PR. Add to `.gitignore` on first run (Step 1).
- **Slug:** current git branch name, or a kebab-case identifier derived from the task if on `main`/`master`
- **Plan file path:** `.critique-loop/<slug>-plan.md` (default — **ephemeral mode**, lives under the gitignored artifact directory, never committed).
  Override to a repo path like `docs/plans/<slug>.md` to use **repo mode** — the plan becomes a first-class committed design doc (skill commits the initial file and each revision as `docs: plan for <slug>` / `docs: revise plan for <slug> (round N)`). Pick repo mode when you want the plan reviewable as part of the PR; pick ephemeral mode when the plan is just scratch for the navigator loop.

Session-id file: `.critique-loop/<slug>.session-id` (created on first call, reused on every resume).

### Navigator adapter

The skill body below refers to two abstract operations: **START-SESSION** (first call, creates the session) and **RESUME-SESSION** (every follow-up call). Each navigator below supplies both, plus its session-id-capture strategy. Pick one set based on the `Navigator CLI` value above.

All navigators accept the same inputs:
- `<PROMPT>` — the prompt text (passed via a heredoc in the actual steps).
- `<OUTPUT_FILE>` — where the navigator's review is written.
- `<SLUG>` — the task slug (used in the session-id path).

#### If Navigator CLI = `codex` (default)

- **Model:** `gpt-5.5` (latest)
- **Reasoning effort:** `xhigh`
- **Sandbox:** `read-only` (navigator reviews; does not write code)

**START-SESSION** (tee the log, grep the session ID after the call):

```bash
codex exec \
  -m gpt-5.5 \
  -c model_reasoning_effort=xhigh \
  --sandbox read-only \
  -o <OUTPUT_FILE> \
  "<PROMPT>" \
  < /dev/null \
  2>&1 | tee .critique-loop/<SLUG>.nav.log

grep -oE 'session id: [0-9a-f-]{36}' .critique-loop/<SLUG>.nav.log \
  | head -1 | awk '{print $3}' > .critique-loop/<SLUG>.session-id
```

**RESUME-SESSION**:

```bash
SID=$(cat .critique-loop/<SLUG>.session-id)
codex exec resume "$SID" \
  -m gpt-5.5 \
  -c model_reasoning_effort=xhigh \
  -c sandbox_mode='"read-only"' \
  -o <OUTPUT_FILE> \
  "<PROMPT>" \
  < /dev/null
```

> **Why `< /dev/null`?** `codex exec` reads from stdin in addition to the positional prompt argument. When the parent agent leaves stdin open (typical for non-interactive harnesses), codex blocks indefinitely on `Reading additional input from stdin...` and never starts the model call — symptom: a codex process alive for many minutes with no log output. Closing stdin is mandatory for non-interactive use.

> **Why `-c sandbox_mode` on resume but `--sandbox` on the first call?** `codex exec` accepts the `--sandbox` flag; `codex exec resume` does NOT — it only takes `-c` config overrides, so the equivalent is `-c sandbox_mode='"read-only"'` (the value is parsed as TOML, hence the inner quotes). Passing `--sandbox` to `resume` errors out with a help dump and no review is generated.

#### If Navigator CLI = `cursor`

- **Model:** `gpt-5.3-codex-xhigh` (latest Codex-series GPT on Cursor, extra-high effort)
- **Mode:** `plan` (read-only; equivalent to Codex's `--sandbox read-only`)
- **Output format:** `text`
- **Headless trust:** `--trust` (required for non-interactive; equivalent to Codex's sandbox approval)

Cursor is cleaner than Codex for session tracking: `cursor-agent create-chat` returns the UUID *before* the first message, so **START-SESSION** and **RESUME-SESSION** use identical invocations; only START creates the chat ID first.

**START-SESSION** (pre-create chat, then run with `--resume`):

```bash
cursor-agent create-chat > .critique-loop/<SLUG>.session-id
SID=$(cat .critique-loop/<SLUG>.session-id)
cursor-agent -p --trust \
  --model gpt-5.3-codex-xhigh \
  --mode plan \
  --output-format text \
  --resume "$SID" \
  "<PROMPT>" \
  > <OUTPUT_FILE>
```

**RESUME-SESSION**:

```bash
SID=$(cat .critique-loop/<SLUG>.session-id)
cursor-agent -p --trust \
  --model gpt-5.3-codex-xhigh \
  --mode plan \
  --output-format text \
  --resume "$SID" \
  "<PROMPT>" \
  > <OUTPUT_FILE>
```

After either START-SESSION, verify `.critique-loop/<SLUG>.session-id` is non-empty before continuing. If empty, surface the raw output to the user and stop.

## Prerequisites

- Navigator CLI is installed and authenticated:
  - **Codex:** `codex --version` succeeds, `codex login` done. Smoke test: `codex exec -m gpt-5.5 --sandbox read-only "reply OK" < /dev/null` prints `OK`. (The `< /dev/null` is required; see the codex adapter note above.)
  - **Cursor:** `cursor-agent --version` succeeds, `cursor-agent login` done. Smoke test: `cursor-agent -p --trust --model gpt-5.3-codex-xhigh --mode plan "reply with exactly: OK"` prints `OK`.
- Current directory is a git repo.
- Current branch is not `main`/`master`. If on `main`, ask the user for a branch name and slug before proceeding.

## Phase 1: Draft the plan

### Step 1: Scope the task, pick the slug, gitignore artifacts

Read the task from the user's request. Explore the repo enough to write a concrete plan (real file paths, existing patterns to match). Don't implement yet.

```bash
SLUG=$(git branch --show-current)
# If on main/master, ask user for a branch + slug
mkdir -p .critique-loop

# .critique-loop/ is local working state — never part of the PR. Ensure it's gitignored.
if ! grep -qxE '\.critique-loop/?' .gitignore 2>/dev/null; then
  echo ".critique-loop/" >> .gitignore
  git add .gitignore
  git commit -m "chore: gitignore .critique-loop artifacts"
fi
```

The `.gitignore` commit is the only critique-loop-related commit that ever lands in the PR **from ephemeral mode**. In repo mode (see Configuration → Plan file path) the plan file itself is also committed (see Step 3).

### Step 2: Write the plan

Write the plan to the configured **Plan file path** (default `.critique-loop/<slug>-plan.md`). If the path is outside `.critique-loop/`, make sure the parent directory exists:

```bash
PLAN_FILE_PATH=".critique-loop/<slug>-plan.md"  # or the user-configured repo path
mkdir -p "$(dirname "$PLAN_FILE_PATH")"
```

Sections:

- **Task** — one paragraph in your own words.
- **Context** — what the repo constrains (existing patterns, relevant files, prior art).
- **Approach** — numbered steps at the level of "edit function X in file Y to do Z".
- **Files to modify** — paths with one-line reason each.
- **Verification** — how you'll know it worked (commands to run, manual checks).
- **Open questions** — things the user or navigator should weigh in on; don't invent answers.

### Step 3: Commit the plan (repo mode only)

If the plan file lives outside `.critique-loop/` (repo mode), commit it:

```bash
if [[ "$PLAN_FILE_PATH" != .critique-loop/* ]]; then
  git add "$PLAN_FILE_PATH"
  git commit -m "docs: plan for <slug>"
fi
```

Skip this step in ephemeral mode — the plan is gitignored working state.

## Phase 2: Navigator reviews the plan

### Step 4: First call (creates the session)

Run **START-SESSION** from the navigator adapter (see Configuration) with:

- `<SLUG>` = the task slug
- `<OUTPUT_FILE>` = `.critique-loop/<slug>-plan-review.md`
- `<PROMPT>`:

```
You are the navigator in an XP pair-programming session. The driver (Claude Code) has written a plan for a task.

Read the plan at `${PLAN_FILE_PATH}` and any referenced code in the repo. Be adversarial — probe for:

- missing edge cases and error paths
- risky assumptions or unstated dependencies
- scope that does not match the stated task (too broad, too narrow, misaligned)
- simpler alternatives the driver may have missed
- verification steps that would not actually catch regressions

Output format:

1. One-sentence summary of your overall take.
2. Numbered list of specific asks. For each: **what** is wrong, **why** it matters, and (if you can) **how** you would address it. Reference file paths and line numbers when relevant.
3. End with EXACTLY one of these lines on its own line, nothing after:
   - `VERDICT: APPROVE` — plan is solid; driver may implement.
   - `VERDICT: CHANGES_REQUESTED` — the numbered asks above must be resolved.
   - `VERDICT: BLOCK` — the plan has a fundamental problem needing human input.

Do not write code. Do not modify files. Review only.
```

Substitute `<slug>` and `${PLAN_FILE_PATH}` literally before passing. Verify `.critique-loop/<slug>.session-id` is non-empty before continuing.

### Step 5: Read the verdict

Read `.critique-loop/<slug>-plan-review.md`. Last line is the verdict.

- **`VERDICT: APPROVE`** → Step 5b (user-approval gate).
- **`VERDICT: CHANGES_REQUESTED`** → Step 6.
- **`VERDICT: BLOCK`** → Step 7.
- **No `VERDICT:` line** → resume with "please restate your review ending with a single `VERDICT:` line". If it fails again, stop.

### Step 5b: Wait for user approval

The navigator approved, but the user has final say before implementation starts. Present a short approval request:

- **Slug** and current branch.
- **One-sentence summary** of what will be implemented.
- **Decisions made during review** — list any user answers from Step 6 that shaped the plan (e.g. "strict `h→m→s` order chosen; bare numbers rejected"). Skip this bullet if there were none.
- **Open questions still unresolved** — read the current plan file's **Open questions** section. List any items not answered by a user decision above. If all are resolved (or the section is empty), skip this bullet. Do NOT silently proceed with unresolved open questions in the plan.
- **Plan file** path (the user can open it to inspect before saying go).
- **Explicit prompt:** "Ready to implement? Say 'go' to proceed, or tell me what to change."

Wait for the user's response before doing anything else.

- **User says "go" / approves** → Phase 3.
- **User requests minor changes** (wording, small scope trim, ergonomic tweaks) → revise the plan file, show the updated summary again. No navigator round — these are user-preference edits, not correctness changes.
- **User requests substantive changes** (new requirement, different approach, new concern the navigator didn't raise) → revise the plan file and go back to Step 6b for another navigator round. The user's change shifted the design; the navigator should re-bless it before Phase 3.
- **User wants to pivot or stop** → stop. The plan file stays on disk (gitignored in ephemeral mode, or committed in repo mode) so they can come back to it later.

Use judgment on the minor-vs-substantive split. If unsure, prefer another navigator round over skipping straight to implementation.

### Step 6: Resolve CHANGES_REQUESTED

Classify each numbered ask:

**Claude resolves directly (no user input):**
- Missing edge case, error path, or failure mode
- Unclear or out-of-order steps
- Wrong file paths, wrong function names, stale references
- Missing test or verification step
- Simpler alternative that preserves the user's stated goal
- Scope trim that stays inside the task

**Surface to the user (Claude lacks the authority):**
- Product decisions ("should this feature exist?", "setting X vs. Y?")
- Architecture tradeoffs with no obvious right answer
- Business rules or domain judgements
- Anything depending on information outside the repo

**Apply:**

1. If any user-surface asks exist, pause. Summarize each for the user in 2–3 bullets (the ask, why it matters, the navigator's suggested options if any). Wait for the user's answer. Fix Claude-resolvable asks in the same revision pass once the user answers.
2. If all asks are Claude-resolvable, fix them in `.critique-loop/<slug>-plan.md` directly.

Write a short note to `.critique-loop/<slug>-driver-response.md`:
- Which numbered asks were addressed and how (one line each).
- Any user answers verbatim.
- Any asks you are pushing back on, and why.

Round number `N` for file naming is tracked by file count in `.critique-loop/` (e.g., look for the highest `-plan-review-<N>.md`). Review files and driver-response are always gitignored regardless of plan mode.

**Repo-mode only:** if the plan lives outside `.critique-loop/`, commit the revision so the navigator in the next round reads the updated file from a real commit:

```bash
if [[ "$PLAN_FILE_PATH" != .critique-loop/* ]]; then
  git add "$PLAN_FILE_PATH"
  git commit -m "docs: revise plan for <slug> (round N)"
fi
```

In ephemeral mode, skip — the navigator reads the working-tree file directly.

### Step 6b: Re-review (resume the same session)

Run **RESUME-SESSION** from the navigator adapter with:

- `<SLUG>` = the task slug
- `<OUTPUT_FILE>` = `.critique-loop/<slug>-plan-review-<N+1>.md` (increment `<N+1>` each round: `-plan-review-2.md`, `-plan-review-3.md`, …)
- `<PROMPT>` (substitute `${PLAN_FILE_PATH}` before passing):

```
I revised the plan based on your review. Updated plan: `${PLAN_FILE_PATH}`. My response to each ask: `.critique-loop/<slug>-driver-response.md`.

Re-review. Note which prior asks are resolved and which remain open. Same output format, end with `VERDICT:`.
```

Go back to Step 5.

### Step 7: Handle BLOCK

Do not loop. Summarize the blocker for the user in 2–4 sentences, include the navigator's suggested direction, stop. Do not re-invoke the navigator until the user responds.

## Phase 3: Implement the approved plan

### Step 8: Record the plan SHA

After the user approves in Step 5b, capture the current HEAD SHA. This is the baseline the implementation will diff against in Phase 4 — everything committed after this point is considered "the implementation":

```bash
git rev-parse HEAD > .critique-loop/<slug>.plan-sha
```

In ephemeral mode the plan is gitignored, so HEAD is the last real commit on the branch. In repo mode the plan's revision commits are included in `<plan-sha>..HEAD`, which is fine — the navigator already approved them in Phase 2, so they don't show up as asks in Phase 4.

### Step 9: Implement

Implement exactly what the approved plan describes. Do not expand scope — if you find something the plan missed, note it for the code-review phase and either (a) fix it if it's a minor oversight in the same area or (b) stop and ask the user if it's a scope question.

Use conventional commits (`feat:`, `fix:`, `refactor:`, etc.), one logical change per commit. Run the project's lint + tests before the last commit. If lint or tests fail and the fix isn't trivial, stop and ask the user.

### Step 10: Write the diff summary

After the final implementation commit, write `.critique-loop/<slug>-diff-summary.md`:

- **Commit range:** `<plan-sha>..HEAD` (use the SHA from Step 8).
- **What changed:** 2–5 bullets, one per significant change.
- **Files touched:** bullet list.
- **Tests added / updated:** bullet list.
- **Anything deferred or out of scope:** bullets (if any).

This file is gitignored working state — do not commit it.

## Phase 4: Navigator reviews the diff

### Step 11: Resume the session for code review

Read the plan SHA first, then interpolate it into the prompt:

```bash
PLAN_SHA=$(cat .critique-loop/<slug>.plan-sha)
```

Run **RESUME-SESSION** from the navigator adapter with:

- `<SLUG>` = the task slug
- `<OUTPUT_FILE>` = `.critique-loop/<slug>-code-review.md`
- `<PROMPT>` (substitute `${PLAN_SHA}` and `${PLAN_FILE_PATH}` before passing):

```
The plan you approved is implemented. Review the diff.

- Plan: `${PLAN_FILE_PATH}`
- Diff summary: `.critique-loop/<slug>-diff-summary.md`
- Commit range: `${PLAN_SHA}..HEAD`

Run `git diff ${PLAN_SHA}..HEAD` and read the changed files. Verify:

(a) Implementation matches the approved plan.
(b) No scope creep beyond what the plan approved.
(c) No introduced bugs, regressions, or missed edge cases.
(d) Tests cover the changes adequately.

Output format: same as before — numbered asks with file:line references where possible, ending with `VERDICT: APPROVE | CHANGES_REQUESTED | BLOCK`.

Do not write code. Do not modify files. Review only.
```

### Step 12: Read the verdict

Read `.critique-loop/<slug>-code-review.md`.

- **`VERDICT: APPROVE`** → Step 14 (done).
- **`VERDICT: CHANGES_REQUESTED`** → Step 13.
- **`VERDICT: BLOCK`** → treat like Phase 2's Step 7: summarize, stop, ask user.

### Step 13: Resolve CHANGES_REQUESTED on code

Same classification rule as Step 6:

- Code-level asks (bugs, missing tests, refactors within the existing scope, edge cases) → Claude fixes in the code directly. Commit with a conventional-commits message (`fix:`, `test:`, etc.).
- Scope / product / architecture asks → surface to the user as in Step 6. Wait for the answer before continuing.

Append a note to `.critique-loop/<slug>-driver-response.md` (same file used in Phase 2; append, don't overwrite):

- Which numbered asks were addressed and how.
- Any user answers verbatim.
- Any pushback and why.

This is gitignored working state — do not commit it. The real code fixes in Step 13 use normal conventional commits (`fix:`, `test:`, etc.) and go in the PR as real work.

Run **RESUME-SESSION** from the navigator adapter with:

- `<SLUG>` = the task slug
- `<OUTPUT_FILE>` = `.critique-loop/<slug>-code-review-<N+1>.md` (increment each round)
- `<PROMPT>` (substitute `${PLAN_SHA}` with the captured SHA before passing):

```
I addressed your code-review asks. New commits are on top of `${PLAN_SHA}..HEAD`. Response notes appended to `.critique-loop/<slug>-driver-response.md`.

Re-review the full diff (`git diff ${PLAN_SHA}..HEAD`). Note which prior asks are resolved and which remain open. Same output format, end with `VERDICT:`.
```

Go back to Step 12.

### Step 14: Approved — report

```
## Critique-loop summary
- Slug: <slug>
- Plan rounds: X
- Code-review rounds: Y
- User questions surfaced: Z (resolved W)
- Plan SHA: <plan-sha>
- Final HEAD: <head-sha>
- Artifacts: .critique-loop/<slug>-*.md
```

The branch is ready for the user to open a PR or merge. PR / merge / push are out of scope for this skill.

## Stopping rules

Bail and ask the user when:

- The navigator returns the same ask in 3 rounds in a row with no sign of converging — your revisions aren't landing; something is miscommunicated.
- Round counter hits 10 in either phase without an APPROVE — at that point escalating is cheaper than iterating.
- Navigator output missing `VERDICT:` twice in a row — the contract isn't holding; surface the raw output.
- The navigator CLI fails (auth expired, network, CLI crash) — report the exact error; do not retry blindly. Do not silently switch to the other navigator CLI to dodge the failure.
- The session-id file is missing or empty after the first call — the first session didn't record properly; do not try to resume.
- The user's answer to a surfaced question is itself ambiguous — re-ask before resuming the navigator.
- During Phase 3, the lint/test suite fails in a way that requires judgement outside the plan (flaky infra, unrelated breakage, architectural conflict) — don't silently rewrite the plan to dodge it.

Never change the model, effort, sandbox mode, or navigator CLI to coerce a different verdict. If the user wants a different configuration, they edit the Configuration section above.

## Review-only flow

Use this when the user has changes already and just wants an adversarial navigator review — no plan, no Claude implementation. Everything from the navigator adapter, the verdict format, the Step 6 classification rule (Claude-resolves vs. user-surface), and the RESUME-SESSION loop applies unchanged.

### Review-only Step R1: Setup

Same as Phase 1 Step 1 — pick the `<slug>` (from the current branch name, or ask if on `main`/`master`), create `.critique-loop/`, and ensure the directory is gitignored:

```bash
SLUG=$(git branch --show-current)
mkdir -p .critique-loop
if ! grep -qxE '\.critique-loop/?' .gitignore 2>/dev/null; then
  echo ".critique-loop/" >> .gitignore
  git add .gitignore
  git commit -m "chore: gitignore .critique-loop artifacts"
fi
```

### Review-only Step R2: Determine the diff range

Ask the user what to review if the intent isn't obvious. Default: the branch's changes versus its merge base with `origin/main`:

```bash
BASE=$(git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main)
REVIEW_RANGE="${BASE}..HEAD"
echo "${REVIEW_RANGE}" > .critique-loop/<slug>.review-range
```

Other shapes the user may want:
- `HEAD` for uncommitted working-tree changes (the navigator reads `git diff HEAD` instead of a range).
- `<sha1>..<sha2>` for a specific commit range.
- `<base>...HEAD` (three-dot) for three-dot merge-base semantics.

Record whichever was chosen in the `review-range` file.

### Review-only Step R3: Navigator reviews the diff

Run **START-SESSION** from the navigator adapter with:

- `<SLUG>` = the task slug
- `<OUTPUT_FILE>` = `.critique-loop/<slug>-code-review.md`
- `<PROMPT>` (substitute `${REVIEW_RANGE}` with the captured range before passing):

```
You are the navigator in a cross-model code review. The driver has changes they want reviewed adversarially before shipping.

Run `git diff ${REVIEW_RANGE}` and read the changed files. Be adversarial — probe for:

- bugs, regressions, missed edge cases
- missing or inadequate tests
- security, performance, or correctness issues
- scope creep or leftover debug code
- simpler alternatives the driver missed

Output format: same as standard critique-loop reviews — numbered asks with file:line refs, ending with `VERDICT: APPROVE | CHANGES_REQUESTED | BLOCK`.

Do not write code. Do not modify files. Review only.
```

### Review-only Step R4: Handle the verdict

Read `.critique-loop/<slug>-code-review.md` and branch on the verdict:

- **`VERDICT: APPROVE`** → summarize rounds and any product decisions surfaced to the user; done. The branch is the user's to push / PR as they see fit.
- **`VERDICT: CHANGES_REQUESTED`** → apply the Step 6 classification:
  - Code-level asks → Claude fixes in the code with conventional commits (`fix:`, `test:`, `refactor:`, etc.).
  - Product / architecture / scope asks → surface to the user, wait for the answer.
  - Append notes to `.critique-loop/<slug>-driver-response.md` as in the full flow (gitignored; do not commit).
  - Then run **RESUME-SESSION** with `<OUTPUT_FILE>` = `.critique-loop/<slug>-code-review-<N+1>.md` (increment each round) and this follow-up prompt (substitute `${REVIEW_RANGE}` before passing):

    ```
    I addressed your code-review asks. New commits are on top of the range. Response notes: `.critique-loop/<slug>-driver-response.md`.

    Re-review the full diff (`git diff ${REVIEW_RANGE}` — note HEAD has moved since your last review). Note which prior asks are resolved and which remain open. Same output format, end with `VERDICT:`.
    ```

  - Loop: go back to the start of Step R4 with the new review file.
- **`VERDICT: BLOCK`** → summarize the blocker for the user in 2–4 sentences, include the navigator's suggested direction, stop. Do not re-invoke the navigator until the user responds.

All stopping rules above still apply (3 identical asks in a row, 5 rounds, missing `VERDICT:` twice, etc.). No user-approval gate — the user is already driving; when the navigator approves, the skill terminates.
