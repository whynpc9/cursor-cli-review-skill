# cursor-cli-review

A skill that uses [Cursor CLI](https://cursor.com/docs/cli/overview) in non-interactive mode for structured code review and development planning.

## What changed in V2

- Added a dedicated **Adversarial Review** mode for pressure-testing design choices and failure modes
- Switched reviewer output from free-form markdown to a strict JSON contract
- Added deterministic review target selection for working tree, branch diff, and explicit file review
- Added large-diff handling: small diffs are inlined, larger diffs use lightweight context plus read-only self-collection
- Added explicit handling for untracked files, large files, binary files, directories, and broken symlinks

## Modes

### Standard Review

1. Resolves the target automatically:
   - working tree when the repo is dirty
   - branch diff when the repo is clean
   - explicit files or base branch when the user specifies them
2. Chooses 1-3 reviewers based on change size:
   - **Correctness**
   - **Design**
   - **Security**
3. Requires every reviewer to return JSON matching [review-output-schema.json](./skills/cursor-cli-review/references/review-output-schema.json)
4. Synthesizes findings into a final verdict: **APPROVE**, **NEEDS ATTENTION**, or **BLOCK**

### Adversarial Review

1. Uses the same target-selection logic as Standard Review
2. Runs a dedicated skeptical reviewer focused on:
   - rollback safety
   - race conditions
   - data loss and corruption
   - trust boundaries
   - hidden failure modes
3. Returns the same structured JSON output and final verdict format

### Development Planning

1. Clarifies the requested goal, context, scope, and constraints
2. Spawns 1-3 planners based on complexity:
   - **Architect**
   - **Implementer**
   - **Risk Analyst**
3. Synthesizes a concrete plan with implementation steps, risks, and milestones

Default model: `opus-4.6`

## Install

Requires [skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add whynpc9/cursor-cli-review-skill
```

Or install to a specific agent:

```bash
npx skills add whynpc9/cursor-cli-review-skill -a cursor
npx skills add whynpc9/cursor-cli-review-skill -a claude-code
```

## Prerequisites

- [Cursor CLI](https://cursor.com/docs/cli/overview) installed and authenticated
- `agent` command available in the shell
- A Git repository when using review modes

## Triggering

This skill is intentionally gated. It should trigger only when the user explicitly mentions `cursor`.

Examples:

```text
use cursor review my changes
cursor review this branch against main
cursor adversarial review this migration
use cursor to pressure-test this auth design
cursor plan how to migrate this service to TypeScript
让cursor帮我做个开发计划
```

It should not trigger on generic requests like `review my code` or `plan this feature`.

## Review output schema

Reviewers must return JSON with:

- `verdict`
- `summary`
- `findings`
- `next_steps`

Each finding must include:

- `severity`
- `title`
- `body`
- `file`
- `line_start`
- `line_end`
- `confidence`
- `recommendation`

The canonical schema lives at [skills/cursor-cli-review/references/review-output-schema.json](./skills/cursor-cli-review/references/review-output-schema.json).

## License

MIT
