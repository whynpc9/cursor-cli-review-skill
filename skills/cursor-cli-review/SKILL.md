---
name: cursor-cli-review
description: >-
  Code review and development planning using Cursor CLI in non-interactive mode. Two modes:
  (1) Review — spawns multiple Cursor agent reviewers to analyze code changes from distinct
  critical perspectives (correctness, design, security); (2) Plan — spawns multiple Cursor
  agent planners to produce a structured development plan from user requirements (architecture,
  implementation, risk analysis). Default model: opus-4.6. ONLY trigger this skill when the
  user explicitly mentions "cursor" in their request — e.g. "用cursor review", "use cursor to
  review", "cursor review my code", "用cursor做计划", "cursor plan this feature", "让cursor帮我
  规划". Do NOT trigger on generic "review my code" or "make a plan" without the word "cursor".
---

# Cursor CLI Code Review & Development Planning

Spawn Cursor CLI agents in non-interactive mode (`agent -p`) to either review code changes
or produce development plans, each from multiple independent perspectives.

**Hard constraint:** All agents MUST run via the Cursor CLI (`agent -p`). Do NOT use
subagents, the Task tool, or any internal delegation mechanism — those run on your own
model within the same session, which defeats the purpose of independent analysis.

## Trigger Gate

This skill requires the user to **explicitly mention "cursor"** (or equivalent like
"cursor cli", "用cursor", "让cursor") in their request. If the user simply says "review
my code" or "make a plan" without referencing cursor, do NOT activate this skill — use
your built-in capabilities instead.

Examples that **should** trigger: "用cursor review一下代码", "use cursor to review my
changes", "cursor review this PR", "让cursor帮我做个开发计划", "cursor plan this feature",
"use cursor cli to plan the implementation".

Examples that **should NOT** trigger: "review my code", "code review", "make a development
plan", "plan this feature" (no mention of cursor).

## Mode Selection

After confirming the user's request mentions cursor, determine which mode to use:

| Trigger phrases | Mode |
|---|---|
| "cursor review", "用cursor review", "cursor检查", "cursor审查代码" | **Review** — go to [Code Review](#code-review) |
| "cursor plan", "用cursor做计划", "cursor规划", "cursor帮我设计方案" | **Plan** — go to [Development Planning](#development-planning) |

If ambiguous, ask the user which mode they want.

---

# Code Review

Spawn reviewers to analyze code changes from multiple critical perspectives. The deliverable
is a synthesized verdict — do NOT make changes to the code.

## Review Step 1 — Determine Scope

Identify what to review from context (recent diffs, referenced files, user message).

Determine the **intent** — what the author is trying to achieve. This is critical:
reviewers judge whether the code *achieves the intent well*, not whether the intent itself
is correct. State the intent explicitly before proceeding.

Collect the diff based on the situation:

```sh
# Uncommitted changes (staged + unstaged)
git diff HEAD

# Feature branch diff against base
git diff main...HEAD

# Staged changes only
git diff --cached

# Specific files
git diff -- path/to/file1 path/to/file2
```

If the diff is very large (>2000 lines), consider scoping to the most critical files or
asking the user which areas to focus on.

## Review Step 2 — Select Reviewers

Assess change size and assign reviewer lenses:

| Size   | Threshold                   | Reviewers                                |
|--------|-----------------------------|------------------------------------------|
| Small  | < 50 lines, 1–2 files      | 1 (Correctness)                          |
| Medium | 50–200 lines, 3–5 files    | 2 (Correctness + Design)                 |
| Large  | 200+ lines or 5+ files     | 3 (Correctness + Design + Security)      |

### Reviewer Lenses

**Correctness** — Logic errors, edge cases, off-by-one bugs, null/undefined handling,
race conditions, incorrect assumptions about data types or shapes, missing error handling,
broken control flow, incorrect return values.

**Design** — Abstraction quality, separation of concerns, naming clarity, code duplication,
adherence to project conventions and patterns, testability, extensibility, unnecessary
complexity, violation of SOLID principles, poor API surface.

**Security** — Input validation, injection risks (SQL, XSS, command), authentication and
authorization gaps, secrets or credentials in code, unsafe deserialization, SSRF/CSRF
vectors, dependency vulnerabilities, insecure defaults, information leakage in error
messages.

## Review Step 3 — Spawn Reviewers via Cursor CLI

Create a temp directory for reviewer output:

```sh
REVIEW_DIR=$(mktemp -d /tmp/cursor-cli-review.XXXXXX)
```

Save the diff to a file so reviewers can access it:

```sh
git diff HEAD > "$REVIEW_DIR/diff.patch"
# or whichever diff command matches the scope from Step 1
```

Spawn each reviewer using `agent -p` in non-interactive print mode. The default model is
`opus-4.6`; override with a different `--model` if the user requests it.

```sh
agent -p "REVIEWER_PROMPT" \
  --model opus-4.6 \
  --mode ask \
  --output-format text \
  --trust \
  > "$REVIEW_DIR/correctness.md" 2>/dev/null
```

Run each reviewer command with `is_background: true` and spawn **all reviewers in parallel**.
Monitor completion by checking if the background processes have finished.

### Reviewer prompt template

Each reviewer gets a single prompt containing:

1. Their assigned lens (full description from the Reviewer Lenses section above)
2. The diff content to review (inline in the prompt, read from the saved diff file)
3. These instructions:

```
You are a code reviewer focusing on [{LENS_NAME}].

Your lens definition:
{LENS_FULL_DESCRIPTION}

Your job is to find real problems, not validate the work. Be specific — cite file
paths, line numbers, and concrete failure scenarios.

Rate each finding by severity:
- critical: Blocks merge — will cause bugs, data loss, or security vulnerabilities
- warning: Should fix before merge — potential issues, poor patterns, subtle bugs
- info: Worth noting — style improvements, minor suggestions

Write findings as a numbered markdown list. For each finding include:
- **[severity]** one-line summary
- File: path and line number(s)
- Issue: detailed description of the problem
- Impact: what could go wrong
- Fix: concrete suggestion (not vague advice)

If you find no issues within your lens, explicitly state that with a brief explanation
of what you checked.

Here is the diff to review:
---
{DIFF_CONTENT}
---
```

Name each output file after the lens: `correctness.md`, `design.md`, `security.md`.

## Review Step 4 — Verify and Synthesize Verdict

Before reading reviewer output, confirm the output files exist and are non-empty:

```sh
ls -la "$REVIEW_DIR"/*.md
```

If any output file is missing or empty, note the failure in the verdict — do not silently
skip a reviewer.

Read each reviewer's output file from `$REVIEW_DIR/`. Deduplicate overlapping findings
(different reviewers may flag the same issue from different angles — merge these into a
single finding noting all relevant perspectives).

Produce a single verdict:

```
## Review Scope
<what was reviewed — branch, files, lines changed, intent>

## Verdict: PASS | NEEDS CHANGES | BLOCK
<one-line summary>

## Findings
<numbered list, ordered by severity (critical → warning → info)>

For each finding:
- **[severity]** Description with file:line references
- Lens: which reviewer raised it
- Impact: what could go wrong
- Fix: concrete action

## What Went Well
<1–3 things the reviewers found no issue with — acknowledge good patterns and decisions>
```

**Verdict logic:**
- **PASS** — no critical or warning findings
- **NEEDS CHANGES** — warning-level findings but no critical issues
- **BLOCK** — one or more critical findings

## Review Step 5 — Render Judgment

After synthesizing the reviewers, apply your own judgment. Reviewers are instructed to be
thorough and may over-flag. Using the stated intent and project context as your frame,
state which findings you accept and which you consider false positives — and why.

Append to the verdict:

```
## Final Assessment
<for each finding: accept or dismiss with a one-line rationale>

## Recommended Actions
<prioritized list of what the author should address before merging>
```

## Review Step 6 — Clean Up

```sh
rm -rf "$REVIEW_DIR"
```

---

# Development Planning

Spawn planners to produce a structured development plan based on user requirements. Each
planner analyzes the codebase and requirements from a different perspective, then their
outputs are synthesized into a single actionable plan. The deliverable is a plan document
— do NOT implement any code.

## Plan Step 1 — Clarify Requirements

Extract the user's requirements from their message. Summarize:

1. **Goal** — what the user wants to build or change (one sentence)
2. **Context** — relevant existing code, constraints, or preferences mentioned
3. **Scope** — is this a new feature, refactor, migration, integration, or bug fix?

If the requirements are vague, ask the user to clarify before proceeding. A good plan
needs clear input.

Gather codebase context for the planners:

```sh
# Project structure overview
find . -type f -not -path '*/node_modules/*' -not -path '*/.git/*' -not -path '*/dist/*' | head -100

# Key config files
cat package.json 2>/dev/null || cat requirements.txt 2>/dev/null || cat go.mod 2>/dev/null || true
```

## Plan Step 2 — Select Planners

Assess requirement complexity and assign planner lenses:

| Complexity | Threshold | Planners |
|---|---|---|
| Simple | Single file or small change, clear approach | 1 (Implementer) |
| Medium | Multiple files, some design decisions needed | 2 (Architect + Implementer) |
| Complex | Cross-cutting concerns, new subsystem, or major refactor | 3 (Architect + Implementer + Risk Analyst) |

### Planner Lenses

**Architect** — High-level system design: component boundaries, data flow, API contracts,
module dependencies, technology choices, patterns to follow (or avoid), how the new work
fits into the existing codebase structure.

**Implementer** — Concrete implementation steps: which files to create or modify, function
signatures, data structures, migration steps, configuration changes, the exact sequence of
work broken into small mergeable increments.

**Risk Analyst** — Potential pitfalls: edge cases, breaking changes, backward compatibility,
performance implications, dependency conflicts, testing gaps, operational concerns
(deployment, rollback), areas where requirements are ambiguous and assumptions may be wrong.

## Plan Step 3 — Spawn Planners via Cursor CLI

Create a temp directory for planner output:

```sh
PLAN_DIR=$(mktemp -d /tmp/cursor-cli-plan.XXXXXX)
```

Save the codebase context to a file for planners to reference:

```sh
echo "=== Project Structure ===" > "$PLAN_DIR/context.txt"
find . -type f -not -path '*/node_modules/*' -not -path '*/.git/*' -not -path '*/dist/*' | head -100 >> "$PLAN_DIR/context.txt"
echo -e "\n=== Key Config ===" >> "$PLAN_DIR/context.txt"
cat package.json 2>/dev/null >> "$PLAN_DIR/context.txt" || true
```

Spawn each planner using `agent -p` in non-interactive print mode with plan mode. The
default model is `opus-4.6`; override with a different `--model` if the user requests it.

```sh
agent -p "PLANNER_PROMPT" \
  --model opus-4.6 \
  --mode plan \
  --output-format text \
  --trust \
  > "$PLAN_DIR/architect.md" 2>/dev/null
```

Run each planner command with `is_background: true` and spawn **all planners in parallel**.

### Planner prompt template

Each planner gets a single prompt containing:

1. Their assigned lens (full description from the Planner Lenses section above)
2. The user's requirements
3. The codebase context
4. These instructions:

```
You are a development planner focusing on [{LENS_NAME}].

Your lens definition:
{LENS_FULL_DESCRIPTION}

Your job is to analyze the requirements and codebase, then produce a thorough plan
from your perspective. Be specific — reference actual files, directories, and patterns
found in the codebase.

Requirements:
---
{USER_REQUIREMENTS}
---

Codebase context:
---
{CODEBASE_CONTEXT}
---

Structure your output as:

## {LENS_NAME} Analysis

### Key Observations
<what you noticed about the codebase relevant to this task>

### Recommendations
<numbered list of specific recommendations from your lens>

### Detailed Plan
<step-by-step breakdown from your perspective, referencing concrete files and code>

Be concrete: name files, functions, and modules. Avoid vague advice like "consider
performance" — instead say exactly what to do and where.
```

Name each output file after the lens: `architect.md`, `implementer.md`, `risk-analyst.md`.

## Plan Step 4 — Verify and Synthesize Plan

Before reading planner output, confirm the output files exist and are non-empty:

```sh
ls -la "$PLAN_DIR"/*.md
```

If any output file is missing or empty, note the failure in the plan — do not silently
skip a planner.

Read each planner's output file from `$PLAN_DIR/`. Merge overlapping observations and
resolve conflicts (where planners disagree, note both perspectives and your recommendation).

Produce a single development plan:

```
## Requirements
<restated goal and scope>

## Architecture Overview
<high-level design from the Architect's analysis — components, data flow, key decisions>

## Implementation Plan
<ordered list of incremental steps from the Implementer's analysis>

For each step:
- **Step N**: one-line summary
- Files: which files to create or modify
- Details: what to do
- Dependencies: which steps must complete first

## Risk Assessment
<findings from the Risk Analyst, ordered by likelihood × impact>

For each risk:
- **[high/medium/low]** Description
- Mitigation: concrete preventive action

## Open Questions
<anything that remains ambiguous and needs user input before implementation>
```

## Plan Step 5 — Render Judgment

After synthesizing, apply your own judgment. Planners may over-engineer or miss simpler
approaches. Evaluate:

- Is the plan proportional to the requirement's complexity?
- Are there simpler alternatives the planners overlooked?
- Do the implementation steps form a logical, incremental sequence?
- Are the risks realistic or exaggerated?

Append to the plan:

```
## Final Recommendation
<your assessment of the plan — what to follow as-is, what to simplify, what to watch out for>

## Suggested Milestones
<break the plan into 2–4 checkpoints where the user can verify progress before continuing>
```

## Plan Step 6 — Clean Up

```sh
rm -rf "$PLAN_DIR"
```

---

## Configuration

| Parameter     | Default      | Description                                         |
|---------------|--------------|-----------------------------------------------------|
| Model         | `opus-4.6`   | Override via user request or env `CURSOR_REVIEW_MODEL` |
| Review mode   | `ask`        | Read-only review mode (reviewers cannot edit code)  |
| Plan mode     | `plan`       | Planning mode (planners analyze but don't implement) |
| Output format | `text`       | Plain text agent output                             |

The `--trust` flag skips workspace trust prompts in non-interactive mode. Review agents use
`--mode ask` (read-only). Plan agents use `--mode plan` (analysis and planning without edits).

## CI/CD Integration

For use in CI pipelines, both modes can be driven non-interactively:

```sh
# Code review in CI
agent -p "Review the changes in this PR for correctness, design, and security issues. \
  The diff is: $(git diff origin/main...HEAD)" \
  --model opus-4.6 \
  --mode ask \
  --output-format text \
  --trust

# Development planning in CI (e.g. for RFC generation)
agent -p "Create a development plan for: <requirement description>" \
  --model opus-4.6 \
  --mode plan \
  --output-format text \
  --trust
```
