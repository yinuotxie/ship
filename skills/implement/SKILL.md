---
name: implement
version: 0.4.0
description: "Use when executing implementation stories from a plan. Codex implements each story, Claude reviews code quality, spec compliance, and convention compliance. Independent stories may be parallelized."
allowed-tools:
  - Bash
  - Read
  - Write
  - Agent
  - AskUserQuestion
  - TaskCreate
  - TaskUpdate
---

# Ship: Implement

Execute implementation stories from a plan. Each story: codex implements
with TDD, claude reviews code quality + spec compliance + convention
compliance, targeted fix on failure. Stories run sequentially by default;
independent stories may be parallelized.

## Hard Rules — What You Must NOT Do

1. **You do NOT write code.** All implementation goes through `codex exec --full-auto`.
   If codex is unavailable, report BLOCKED — do not implement yourself.
2. **You do NOT review code yourself.** All reviews go through a fresh `Agent` tool
   dispatch. Reading the diff yourself biases judgment — dispatch a reviewer who
   has never seen the implementation context.
3. **You do NOT run tests to "quickly check".** Tests are run by codex (during
   implementation) and by the reviewer agent (during review).

If you catch yourself writing code, reading diffs, or running tests directly,
STOP — you are violating the separation of concerns that makes this pipeline reliable.

## Checklist

You MUST create a task for each of these steps and complete them in order:

1. **Read spec + plan** — read spec.md for acceptance criteria, plan.md for stories
2. **Detect tooling** — find test command, extract code conduct from CLAUDE.md/AGENTS.md
3. **Per-story loop** — for each story: record start SHA → codex implements → record end SHA → verify commit exists → reviewer checks → verdict
4. **Cross-story regression** — run full test suite after all stories pass
5. **Report** — completion status with concerns log

Mark each task in_progress when starting, completed when done.

---

## Design Principles

1. **Fresh Reviewer** — Dispatch a new Reviewer agent per review. Never
   reuse a prior Reviewer — accumulated context biases judgment.
2. **Fix Forward** — On FAIL, dispatch a targeted fix for the specific
   issues found. Never re-implement the entire story.
3. **Investigate First** — When codex reports NEEDS_CONTEXT or uncertainty,
   provide context or break the story smaller. Never force codex to guess.

## Team Composition

### Implementor (codex)

- **Executor:** `codex exec --full-auto`
- **Model selection:** Use `-m <model>` based on story complexity:
  - Mechanical (isolated function, 1-2 files) → fast model
  - Integration (multi-file coordination) → standard model
  - Architecture (design judgment, broad codebase) → most capable model

### Reviewer (claude)

- **Executor:** `Agent` tool (fresh per review)
- **Tools:** Read, Grep, Glob, Bash (read-only)

---

## Input

The caller provides a task directory path containing:
- `plan/spec.md` — acceptance criteria and requirements
- `plan/plan.md` — ordered implementation stories

If invoked standalone (no task dir from caller), use AskUserQuestion to
prompt the user for a task dir or spec + plan paths.

## Process

1. Read `plan/spec.md` — this is the acceptance criteria reference.
2. Read `plan/plan.md` — extract all implementation step sections.
   Accept any heading format: `## Story N`, `## Step N`, `## N. Title`,
   or numbered/bulleted lists of implementation steps. Normalize them as
   ordered stories for the loop.
3. Detect the repo's test command by inspecting the project root:
   - `Makefile` → `make test`
   - `package.json` → `npm test` or `pnpm test` or `yarn test`
   - `pytest.ini` / `pyproject.toml` / `setup.cfg` → `pytest`
   - `go.mod` → `go test ./...`
   - `Cargo.toml` → `cargo test`
   - `mix.exs` → `mix test`
   - `build.gradle` / `pom.xml` → `./gradlew test` or `mvn test`
   - `.csproj` / `.sln` → `dotnet test`
   - `Gemfile` → `bundle exec rspec` or `bundle exec rake test`
   Also check for repo-specific test scripts (e.g. `scripts/test.sh`,
   CI config test steps) and CLAUDE.md/AGENTS.md for documented test
   commands. If none found, use AskUserQuestion. Record as `TEST_CMD`.
4. Extract the repo's code conduct: read `CLAUDE.md`, `AGENTS.md`, and
   any local `AGENTS.md` or `CLAUDE.md` in directories the plan touches.
   Look for coding style rules, naming conventions, error handling
   patterns, test conventions, and hard rules. Also inspect any lint or
   formatter configs in the project root for implicit conventions. If no
   documented conventions exist, sample 2-3 existing files in the same
   directory as the planned changes to infer patterns. Record as
   `CODE_CONDUCT`.
5. Create a task for each story via TaskCreate. Mark in_progress when
   starting, completed when review passes.
6. Run the per-story loop sequentially by default. Stories with no file
   dependencies may be parallelized at your discretion.
7. After all stories: run `TEST_CMD` one final time to catch cross-story
   regressions. If tests fail, dispatch a targeted fix and re-verify.
8. Report completion status with concerns log.

---

## Per-Story Loop

```
For each story i/N:
  1. Record STORY_START_SHA = current HEAD
  2. Implementor implements (codex exec) → commit(s)
  3. Record STORY_HEAD_SHA = current HEAD
  4. Reviewer checks code + spec (Agent, fresh) → verdict
     PASS → next story
     PASS_WITH_CONCERNS → record concerns, next story
     FAIL → targeted fix (max 2 rounds) → re-review
```

### Step A: Implement

Record `STORY_START_SHA`:
```bash
git rev-parse HEAD
```

Dispatch codex exec with the Implementor prompt:

```bash
codex exec "You are implementing story <i>/<N>.
Your code will be reviewed.

## Story <i>/<N>: <title>
<full story text from plan.md>

## Acceptance Criteria
<criteria from spec.md that apply to this story>

## Prior Stories Completed
<for each prior story: title, files changed, commit range>

## Code Conduct
<CODE_CONDUCT — extracted conventions for this repo>

Follow these conventions strictly. Deviating from them is a review
failure even if the code works. If Code Conduct specifies a commit
message format, use it. Otherwise use Conventional Commits.

## Instructions

Follow the TDD cycle:
1. Write a failing test that captures the story requirement (Red)
2. Write the minimal code to make the test pass (Green)
3. Verify all existing tests still pass: <TEST_CMD>
4. Commit — this is MANDATORY, do not skip:
   ```
   git add -A
   git commit -m "<type>(<scope>): <description>"
   ```
   If you do not run git commit, your work is lost and the story fails.

## Code Organization

- If the plan defines file structure, follow it
- Each file should have one clear responsibility
- If a file you are creating is growing beyond the plan's intent, stop and
  report DONE_WITH_CONCERNS — do not split files on your own
- If an existing file you are modifying is already large or tangled, work
  carefully and note it as a concern in your report

## Self-Review Before Committing

Before committing, check your own work:

- **Completeness:** Did you implement every requirement in this story?
  Any edge cases you missed?
- **Quality:** Are names clear? Is the code the simplest thing that works?
- **Discipline:** Did you build ONLY what the story asks? No gold-plating,
  no unasked-for abstractions?
- **Testing:** Do your tests verify actual behavior? Would a failing test
  catch a real regression?

If you find issues, fix them before committing.

## When Stuck

If something is unexpected or unclear, investigate first — read the
relevant code, check tests, understand context. Do not guess.

STOP and report if:
- Investigation does not resolve your uncertainty
- The task requires architectural decisions with multiple valid approaches
- The story involves restructuring code the plan didn't anticipate
- The codebase state does not match what the story assumes (e.g. a prior
  story changed the target file significantly)

Report NEEDS_CONTEXT with what you found, or BLOCKED with what you tried.

## Report Format

End your output with exactly one of these status lines:
DONE — implemented and committed
DONE_WITH_CONCERNS — implemented, but: <specific concerns>
BLOCKED — cannot complete: <what's blocking and what you tried>
NEEDS_CONTEXT — missing: <specific information needed>

Then list: commit SHA and any concerns." --full-auto
```

After Implementor returns:
1. Record `STORY_HEAD_SHA=$(git rev-parse HEAD)`
2. If `STORY_HEAD_SHA == STORY_START_SHA` and status is DONE → treat as
   BLOCKED (claimed done but made no commits).
3. If Implementor reported BLOCKED or NEEDS_CONTEXT → escalate to caller.
4. If Implementor reported DONE_WITH_CONCERNS → log concerns, proceed to
   review normally.
5. Otherwise → proceed to Step B.

### Step B: Review

Dispatch a fresh Agent with the following prompt:

```
You are reviewing the changes for story <i>/<N>.
Review for: code quality, spec compliance, and convention compliance.

## Changes

Run `git diff <STORY_START_SHA>..<STORY_HEAD_SHA>` to see the diff.

## Tests

Run `<TEST_CMD>`. If tests fail, verdict is FAIL regardless of code
quality or spec compliance.

### Code Quality
- Is the code clear and maintainable?
- Are there bugs, logic errors, or obvious edge cases missed?
- Are tests meaningful (not just testing mocks)?

### Spec Compliance
- Does the implementation satisfy the story requirements and acceptance
  criteria below?
- Did the implementor build anything NOT required by the story?
  Extra features or scope creep = FAIL.

<relevant acceptance criteria from spec.md>

### Convention Compliance
Does the code follow the repo's coding conventions?

<CODE_CONDUCT — same conventions given to Implementor>

Check:
- Naming, error handling, file organization, test patterns
- If a convention is not documented, check existing code in the same
  directory for implicit patterns and enforce consistency

Convention violations are FAIL even if the code works correctly.

### Story Requirements
<full story text from plan.md>

## Verdict

Reply with exactly one of:

PASS — tests pass, spec met, conventions followed. Code can proceed.

PASS_WITH_CONCERNS — code can proceed, but these points need attention
in later review: <concerns with file:line references>

FAIL — code must not proceed until fixed: <issues, each with:>
  - What's wrong (quality / spec / convention violation)
  - Where (file:line)
  - How to fix it
```

After Reviewer returns, read the verdict:
- **PASS** → record story as complete, move to next story.
- **PASS_WITH_CONCERNS** → append concerns to `concerns.md` in task dir,
  move to next story.
- **FAIL** → proceed to Step C (targeted fix). Max 2 rounds.
  If 2 rounds exhausted and still FAIL → escalate as BLOCKED.
- **No recognized verdict** → re-dispatch a fresh Reviewer once.
  If still unparseable → treat as FAIL.

### Step C: Targeted Fix

On FAIL, first verify repo state is clean:

```bash
git rev-parse HEAD   # confirm current HEAD
git status --short   # check for uncommitted changes
```

If there are uncommitted partial changes from a crashed prior attempt,
stash or discard them before proceeding (warn the user).

Dispatch a targeted fix via codex exec:

```bash
codex exec "Fix the following issues found by a code reviewer.

## Issues to Fix
<Reviewer's FAIL findings, verbatim>

## Code Conduct
<CODE_CONDUCT>

## Rules
- Fix ONLY the issues listed above. Do not refactor or improve other code.
- Follow the Code Conduct conventions above.
- Run the full test suite after fixes: <TEST_CMD>
- If a fix requires a new test, add it.
- Commit using the commit format from Code Conduct, or Conventional Commits.

Do NOT re-implement the story. Make surgical fixes." --full-auto
```

After fix commits:
1. Update `STORY_HEAD_SHA=$(git rev-parse HEAD)`
2. Return to **Step B** with fresh Reviewer using updated commit range.

---

## Cross-Story Context

After each story completes (PASS or PASS_WITH_CONCERNS), record a summary:

```
Story <i>: "<title>"
  Commits: <STORY_START_SHA>..<STORY_HEAD_SHA> (<N> commits)
  Files: <list of ALL files changed across all commits in range>
  Concerns: <any PASS_WITH_CONCERNS notes, or "none">
```

Use `git diff --name-only <STORY_START_SHA>..<STORY_HEAD_SHA>` to get the
complete file list including any targeted fix commits.

Pass this summary to the next story's Implementor prompt in the
"Prior Stories Completed" section to prevent duplicate work, file
conflicts, and context gaps.

---

## Progress Reporting

Use `[Implement]` prefix for all status output:

```
[Implement] Starting — 5 stories, test cmd: make test
[Implement] Story 1/5: "Add user model" → implementing...
[Implement] Story 1/5: implemented (3 files, 1 commit). Reviewing...
[Implement] Story 1/5: PASS.
[Implement] Story 2/5: "Wire API endpoints" → implementing...
[Implement] Story 2/5: implemented (4 files, 1 commit). Reviewing...
[Implement] Story 2/5: FAIL — missing input validation. Fixing (1/2)...
[Implement] Story 2/5: fix applied (1 commit). Re-reviewing...
[Implement] Story 2/5: PASS (2 rounds).
[Implement] Story 3/5: "Add auth middleware" → implementing...
[Implement] Story 3/5: implemented (2 files, 1 commit). Reviewing...
[Implement] Story 3/5: PASS_WITH_CONCERNS — edge case in token expiry.
...
[Implement] All 5 stories complete. 1 concern recorded.
```

Rules:
- Every status line starts with `[Implement]`
- Show phase transitions: implementing → reviewing → verdict
- Include counts: story 2/5, fix round 1/2
- On PASS_WITH_CONCERNS: include the concern summary
- Never go silent for more than one dispatch without a status update

## Error Handling

Error cases are evaluated in order — first specific match wins.

| Condition | Action |
|-----------|--------|
| Reviewer FAIL, rounds < 2 | Targeted fix (Step C) → fresh re-review |
| Reviewer FAIL, rounds exhausted | Escalate BLOCKED with full findings |
| Reviewer malformed output | Re-dispatch fresh Reviewer once, then treat as FAIL |
| Implementor BLOCKED or NEEDS_CONTEXT | Escalate to caller with report |
| Implementor DONE_WITH_CONCERNS | Log concerns, proceed to Reviewer normally |
| codex exec resource limit (exit 124/137) | Story too large — break it or reduce scope, escalate BLOCKED |
| codex exec other crash (exit ≠ 0) | Check HEAD + working tree for partial changes; stash/discard if dirty; retry once; if still fails → BLOCKED |
| Agent dispatch failure | Retry once, then BLOCKED |

## Completion Status

Report one of:

- **DONE** — all stories implemented, reviewed, and committed.
  Include: total stories, total review rounds, concerns count.
- **DONE_WITH_CONCERNS** — all stories pass, but concerns were recorded.
  Include: concerns.md path for downstream review.
- **BLOCKED** — a story failed after max retries or hit an unresolvable issue.
  Include: which story, what failed, what was tried.
- **NEEDS_CONTEXT** — missing information. Include: what's needed, from whom.

<Bad>
- Writing code yourself instead of dispatching codex exec (violates separation of concerns)
- Reading the diff yourself instead of dispatching a fresh Agent reviewer (biases judgment)
- Running tests yourself instead of letting codex/reviewer handle it
- Falling back to "I'll just do it myself" when codex is slow or unavailable (report BLOCKED instead)
- Skipping review because "codex's self-review looked good"
- Full re-implementation on FAIL instead of targeted fix (wasteful, risky)
- Letting codex modify tests to make them pass instead of fixing code
- Retrying after crash without checking HEAD/working tree for partial changes
- Not using TaskCreate/TaskUpdate to track per-story progress
</Bad>
