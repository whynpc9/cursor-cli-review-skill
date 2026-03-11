# cursor-cli-review

A code review skill that spawns [Cursor CLI](https://cursor.com/docs/cli/overview) agents in non-interactive mode to review code changes from multiple critical perspectives.

## What it does

When triggered, the skill:

1. Collects the relevant diff (uncommitted changes, branch diff, or specified files)
2. Spawns 1–3 independent Cursor CLI reviewers in parallel using `agent -p`
3. Each reviewer analyzes the code from a distinct lens — **Correctness**, **Design**, **Security**
4. Synthesizes findings into a single verdict: **PASS**, **NEEDS CHANGES**, or **BLOCK**

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

- [Cursor CLI](https://cursor.com/docs/cli/overview) installed and authenticated (`agent` command available)
- Git repository with changes to review

## Usage

Once installed, the skill triggers when you ask your coding agent to review code:

```
review my changes
```

```
code review the current diff
```

```
review this PR for issues
```

## License

MIT
