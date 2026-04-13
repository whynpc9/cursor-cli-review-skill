---
name: cursor-cli-review
description: >-
  Code review and development planning using Cursor CLI in non-interactive mode. Three modes:
  (1) Standard Review — spawns 1-3 Cursor reviewers for correctness, design, and security;
  (2) Adversarial Review — runs a dedicated skeptical reviewer that pressure-tests the chosen
  approach and failure modes; (3) Plan — spawns 1-3 Cursor planners for architecture,
  implementation, and risk analysis. Default model: opus-4.6. ONLY trigger this skill when
  the user explicitly mentions "cursor" in their request — for example "use cursor review",
  "cursor adversarial review", "用cursor review", "让cursor帮我做计划". Do NOT trigger on
  generic "review my code" or "make a plan" without the word "cursor".
---

# Cursor CLI Review And Planning

Spawn Cursor CLI agents in non-interactive mode (`agent -p`) to either review code changes or
produce development plans, each from clearly separated perspectives.

Hard constraint:
- All delegated analysis in this skill MUST run via Cursor CLI using `agent -p`.
- Do NOT use subagents, Task, or your own internal delegation as a substitute.
- Review modes are read-only. Do not patch code while performing the review.

## Trigger Gate

This skill requires the user to explicitly mention `cursor`, `cursor cli`, `用cursor`, or
`让cursor` in the request.

Examples that should trigger:
- `use cursor review my changes`
- `cursor review this branch`
- `cursor adversarial review this auth refactor`
- `用cursor review 一下`
- `让cursor帮我做个开发计划`

Examples that should not trigger:
- `review my code`
- `do a code review`
- `plan this feature`
- `make a development plan`

If the user did not mention Cursor, do not activate this skill.

## Mode Selection

After confirming the trigger gate, choose exactly one mode:

| Trigger phrases | Mode |
|---|---|
| `cursor review`, `用cursor review`, `cursor审查代码`, `cursor code review` | [Standard Review](#standard-review) |
| `cursor adversarial review`, `cursor challenge review`, `用cursor挑刺`, `cursor pressure-test this`, `cursor question this design` | [Adversarial Review](#adversarial-review) |
| `cursor plan`, `用cursor做计划`, `cursor规划`, `让cursor帮我设计方案` | [Development Planning](#development-planning) |

If the request says `cursor review` but also asks to challenge the chosen direction, assumptions,
tradeoffs, or failure modes, prefer Adversarial Review.

## Preconditions

Before spawning any Cursor agent:

1. Confirm the `agent` command is available.
2. Confirm the current directory is the intended repository or project root.
3. For review modes, confirm the repo has reviewable work:
   - dirty working tree, or
   - branch diff against an explicit or inferred base, or
   - explicitly referenced files.

If Cursor CLI is missing or unauthenticated, stop and tell the user what is missing.

---

# Standard Review

Run 1-3 independent Cursor reviewers, each with a distinct critical lens, then synthesize the
results into one final judgment.

## Review Step 1 — Determine Intent

State the author intent before collecting the diff. Reviewers should judge whether the change
implements the apparent intent safely and coherently, not whether the product requirement itself
is good.

Summarize the intent in one sentence using the user request plus the observed diff.

## Review Step 2 — Resolve Review Target

Choose the target deterministically. Do not improvise an arbitrary diff command.

### Supported targets

- Working tree review
- Branch review against a base ref
- Explicit file-focused review

### Resolution rules

1. If the user explicitly names files, scope the review to those files first.
2. If the user provides `--base <ref>` or clearly names a base branch, use branch review.
3. If the user explicitly asks for working tree or current changes, use working tree review.
4. Otherwise:
   - If the repository is dirty, review the working tree.
   - If the repository is clean, review the current branch against the detected default branch.

### Default branch detection

Try these in order:

1. `refs/remotes/origin/HEAD`
2. local or remote `main`
3. local or remote `master`
4. local or remote `trunk`

If the repo is clean and no base branch can be inferred, ask the user for the base ref instead
of guessing.

### Recommended commands

```sh
# Repository sanity
git rev-parse --show-toplevel
git branch --show-current
git status --short --untracked-files=all

# Working tree file sets
git diff --cached --name-only
git diff --name-only
git ls-files --others --exclude-standard

# Branch review
git merge-base HEAD <base>
git diff --name-only <base>...HEAD
git diff --stat <base>...HEAD
git diff <base>...HEAD
```

## Review Step 3 — Collect Review Context

Do not always inline the full diff. Choose the input strategy based on size.

### Thresholds

- `MAX_INLINE_FILES = 2`
- `MAX_INLINE_DIFF_BYTES = 262144` (256 KiB)
- `MAX_UNTRACKED_BYTES = 24576` (24 KiB per untracked text file)

### Input modes

#### `inline-diff`

Use inline diff when both are true:

- changed files `<= 2`
- combined diff size `<= 256 KiB`

Provide:

- `git status --short --untracked-files=all`
- staged diff
- unstaged diff
- branch diff if branch review
- contents of untracked text files within the size limit

#### `self-collect`

Use lightweight context when either threshold is exceeded.

Provide only:

- `git status --short --untracked-files=all`
- diff stat
- changed file list
- commit log for branch review
- contents of untracked text files within the size limit

Then explicitly instruct the Cursor reviewer to inspect the target diff itself using read-only
git commands before finalizing findings.

### Untracked file handling

Treat untracked files as reviewable input.

- Include small text files.
- Skip directories.
- Skip binary files.
- Skip broken symlinks or unreadable paths.
- If a file exceeds `MAX_UNTRACKED_BYTES`, note that it was skipped due to size.

## Review Step 4 — Select Reviewers

Size the reviewer set from the change size:

| Size | Threshold | Reviewers |
|---|---|---|
| Small | under 50 changed lines or 1-2 files | 1 reviewer: Correctness |
| Medium | 50-200 changed lines or 3-5 files | 2 reviewers: Correctness + Design |
| Large | over 200 changed lines or more than 5 files | 3 reviewers: Correctness + Design + Security |

### Reviewer lenses

**Correctness**
- Logic errors
- edge cases
- null or undefined handling
- off-by-one behavior
- race conditions
- broken control flow
- invalid data-shape assumptions
- missing error handling

**Design**
- poor abstraction boundaries
- unnecessary complexity
- duplication
- weak naming when it obscures behavior
- poor testability
- extension hazards
- mismatch with repository conventions

**Security**
- input validation gaps
- auth or authorization issues
- unsafe command, SQL, or HTML handling
- trust-boundary violations
- secret leakage
- insecure defaults
- failure modes that expose sensitive state

Do not ask any reviewer to give style-only feedback.

## Review Step 5 — Reviewer Output Contract

Every Cursor reviewer must return only valid JSON following the schema in
[references/review-output-schema.json](./references/review-output-schema.json).

Top-level shape:

```json
{
  "verdict": "approve | needs-attention",
  "summary": "short ship/no-ship assessment",
  "findings": [
    {
      "severity": "critical | high | medium | low",
      "title": "short finding title",
      "body": "grounded explanation",
      "file": "relative/path.ts",
      "line_start": 10,
      "line_end": 14,
      "confidence": 0.82,
      "recommendation": "concrete remediation"
    }
  ],
  "next_steps": ["..."]
}
```

Rules for every reviewer:

- Return JSON only. No markdown fences. No preamble.
- Report only material findings.
- Every finding must cite a concrete file and line range.
- If a conclusion depends on inference, say so in `body` and lower `confidence`.
- Prefer one strong finding over several weak ones.
- If there are no material issues within the lens, return `approve` with an empty `findings` array.

## Review Step 6 — Reviewer Prompt Template

Use a prompt of this shape for each standard reviewer:

```text
You are a code reviewer focusing on [{LENS_NAME}].

Your lens definition:
{LENS_FULL_DESCRIPTION}

Author intent:
{INTENT_SUMMARY}

Review target:
{TARGET_LABEL}

Review input mode:
{INPUT_MODE}

Instructions:
- Find real implementation risks, not style nits.
- Ground every finding in the provided repository context or in read-only git inspection.
- If the input mode is self-collect, inspect the relevant diff yourself with read-only git commands before finalizing.
- Return only valid JSON matching the provided schema.
- Use approve only if you cannot support any material finding from this lens.

Repository context:
---
{REVIEW_INPUT}
---
```

Name output files after the lens, for example `correctness.json`, `design.json`, `security.json`.

## Review Step 7 — Verify And Synthesize

Before synthesizing:

1. Verify every expected output file exists and is non-empty.
2. Parse every file as JSON.
3. If any reviewer output is invalid JSON, report that explicitly instead of silently skipping it.
4. Deduplicate overlapping findings across lenses.

Synthesis rules:

- Merge duplicate findings when they point to the same file, overlapping lines, and the same root issue.
- Preserve all relevant lenses that independently raised the issue.
- Sort findings by severity first, then confidence.

Final judgment mapping:

- `APPROVE` when no reviewer reports a material finding
- `NEEDS ATTENTION` when there are only `high`, `medium`, or `low` findings
- `BLOCK` when any `critical` finding remains after synthesis

Render the final result as:

```text
## Review Scope
<target, file count, input mode, intent>

## Verdict
APPROVE | NEEDS ATTENTION | BLOCK

## Findings
<numbered list ordered by severity>

For each finding:
- Severity
- File and lines
- Lenses
- Why it matters
- Recommended fix
- Confidence

## Reviewer Failures
<any missing or unparsable reviewer outputs>

## Final Assessment
<which findings you accept, dismiss, or downgrade, with one-line rationale>

## Recommended Actions
<prioritized next steps>
```

Apply your own judgment after reading the reviewers. Do not treat every reviewer finding as
automatically correct.

---

# Adversarial Review

Run a single dedicated Cursor reviewer whose job is to challenge whether the chosen approach
should ship at all.

Use this mode when the user wants to pressure-test design choices, tradeoffs, invariants,
rollback behavior, concurrency assumptions, or hidden failure modes.

## Adversarial Step 1 — Reuse Target Resolution And Context Collection

Use the exact same target resolution and context collection rules as Standard Review.

## Adversarial Step 2 — Use A Skeptical Operating Stance

The reviewer should actively try to disprove the change.

Prioritize:

- auth, permissions, and trust boundaries
- data loss, duplication, corruption, and irreversible writes
- rollback safety, retries, and idempotency
- race conditions and stale-state assumptions
- timeout, null, and degraded-dependency behavior
- schema drift and compatibility risk
- observability gaps that would hide failure

Do not waste time on naming, formatting, or cosmetic feedback.

## Adversarial Step 3 — Optional User Focus

If the user named a concern, weight it heavily. Examples:

- `challenge whether this caching design is safe`
- `look for race conditions`
- `pressure-test the migration rollback path`

Still report any other material issue you can defend.

## Adversarial Step 4 — Output Contract

Use the same JSON schema as Standard Review:

- `verdict`
- `summary`
- `findings`
- `next_steps`

Return `needs-attention` if there is any material risk worth blocking on.
Return `approve` only if you cannot support any substantive adversarial finding from the
provided context.

## Adversarial Step 5 — Prompt Template

Use a prompt of this shape:

```text
You are performing an adversarial software review.
Your job is to break confidence in the change, not to validate it.

Target:
{TARGET_LABEL}

Author intent:
{INTENT_SUMMARY}

User focus:
{USER_FOCUS_OR_NONE}

Operating stance:
- Default to skepticism.
- Assume the change can fail in subtle, expensive, user-visible ways until the evidence says otherwise.
- If something only works on the happy path, treat that as a real weakness.

Method:
- Look for violated invariants, missing guards, unhandled failure paths, and assumptions that break under retries, concurrency, or partial completion.
- If the input mode is self-collect, inspect the target diff yourself using read-only git commands before finalizing.
- Report only material findings.
- Return only valid JSON matching the provided schema.

Repository context:
---
{REVIEW_INPUT}
---
```

## Adversarial Step 6 — Render Result

Render the result in the same final format as Standard Review, but label the mode as
`Adversarial Review`.

If the adversarial reviewer returns no findings, say that explicitly and keep the conclusion
short.

---

# Development Planning

Spawn Cursor planners to produce a concrete development plan. The deliverable is a plan document.
Do not implement code in this mode.

## Plan Step 1 — Clarify Requirements

Summarize:

1. Goal
2. Context
3. Scope
4. Constraints

If the requirements are too vague to plan responsibly, ask the user to clarify.

## Plan Step 2 — Gather Codebase Context

Collect lightweight repository context before spawning planners:

```sh
find . -type f -not -path '*/node_modules/*' -not -path '*/.git/*' -not -path '*/dist/*' | head -100
cat package.json 2>/dev/null || cat requirements.txt 2>/dev/null || cat go.mod 2>/dev/null || true
git status --short --untracked-files=all
```

Read additional files only when they are clearly relevant to the requested plan.

## Plan Step 3 — Select Planners

| Complexity | Threshold | Planners |
|---|---|---|
| Simple | single file or obvious local change | Implementer |
| Medium | multiple files with some design choices | Architect + Implementer |
| Complex | cross-cutting change, new subsystem, migration, risky refactor | Architect + Implementer + Risk Analyst |

### Planner lenses

**Architect**
- component boundaries
- data flow
- API contracts
- module placement
- how the work fits the existing codebase

**Implementer**
- exact files to modify
- function or module changes
- dependency order
- small mergeable steps

**Risk Analyst**
- migration hazards
- backward compatibility
- operational risk
- testing gaps
- ambiguity in the requirements

## Plan Step 4 — Planner Prompt Template

Use a prompt of this shape for each planner:

```text
You are a development planner focusing on [{LENS_NAME}].

Your lens definition:
{LENS_FULL_DESCRIPTION}

Requirements:
---
{USER_REQUIREMENTS}
---

Codebase context:
---
{CODEBASE_CONTEXT}
---

Return a concrete plan from your lens. Be specific:
- cite actual files, directories, modules, and patterns
- avoid vague advice
- propose incremental steps rather than one giant rewrite
```

## Plan Step 5 — Synthesize The Final Plan

Merge the planner outputs into:

```text
## Requirements
<restated goal and scope>

## Architecture Overview
<high-level design and key decisions>

## Implementation Plan
<ordered incremental steps with files and dependencies>

## Risks
<ordered by impact x likelihood, with mitigations>

## Open Questions
<ambiguities that still need a decision>

## Final Recommendation
<what to follow, what to simplify, what to watch out for>

## Suggested Milestones
<2-4 checkpoints>
```

Apply your own judgment after synthesis. Simplify over-engineered plans when appropriate.

---

## Configuration

| Parameter | Default | Description |
|---|---|---|
| Model | `opus-4.6` | Override if the user requests a different Cursor model |
| Review mode | `ask` | Read-only Cursor review mode |
| Plan mode | `plan` | Cursor planning mode |
| Output format | `text` | Use plain text transport, but reviewer content must be JSON |

Use `--trust` in non-interactive mode to skip trust prompts when appropriate.

## Operational Notes

- Spawn all reviewers or planners in parallel when there is more than one.
- Store reviewer output in a temp directory and clean it up after synthesis.
- Never silently ignore a missing reviewer result.
- Never claim a finding is grounded if the supporting diff or file was not actually inspected.
