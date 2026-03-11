---
name: cursor-cli-review
description: >-
  Code review using Cursor CLI in non-interactive mode. Spawns multiple Cursor agent
  reviewers to analyze code changes from distinct critical perspectives (correctness,
  design, security). Default model: opus-4.6. Use this skill when the user asks for
  "code review", "review my changes", "review this PR", "review my code", "cursor review",
  or wants automated multi-perspective code analysis on diffs or staged changes.
---

# Cursor CLI Code Review

Spawn Cursor CLI agents in non-interactive mode (`agent -p`) to review code changes from
multiple critical perspectives. The deliverable is a synthesized verdict — do NOT make
changes to the code.

**Hard constraint:** Reviewers MUST run via the Cursor CLI (`agent -p`). Do NOT use
subagents, the Task tool, or any internal delegation mechanism as reviewers — those run
on your own model within the same session, which defeats the purpose of independent review.

## Step 1 — Determine Scope

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

## Step 2 — Select Reviewers

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

## Step 3 — Spawn Reviewers via Cursor CLI

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

## Step 4 — Verify and Synthesize Verdict

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

## Step 5 — Render Judgment

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

## Step 6 — Clean Up

```sh
rm -rf "$REVIEW_DIR"
```

## Configuration

| Parameter     | Default      | Description                                         |
|---------------|--------------|-----------------------------------------------------|
| Model         | `opus-4.6`   | Override via user request or env `CURSOR_REVIEW_MODEL` |
| Mode          | `ask`        | Read-only review mode (reviewers cannot edit code)  |
| Output format | `text`       | Plain text reviewer output                          |

The `--trust` flag skips workspace trust prompts in non-interactive mode. The `--mode ask`
flag ensures reviewers operate in read-only mode and cannot modify code.

## CI/CD Integration

For use in CI pipelines, the entire review can be driven non-interactively:

```sh
# Run a single-pass review in CI
agent -p "Review the changes in this PR for correctness, design, and security issues. \
  The diff is: $(git diff origin/main...HEAD)" \
  --model opus-4.6 \
  --mode ask \
  --output-format text \
  --trust
```
