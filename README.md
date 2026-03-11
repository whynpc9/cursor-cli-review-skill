# cursor-cli-review

A skill that spawns [Cursor CLI](https://cursor.com/docs/cli/overview) agents in non-interactive mode for multi-perspective code review and development planning.

## What it does

Two modes, automatically selected based on your request:

### Code Review

1. Collects the relevant diff (uncommitted changes, branch diff, or specified files)
2. Spawns 1–3 independent Cursor CLI reviewers in parallel using `agent -p`
3. Each reviewer analyzes the code from a distinct lens — **Correctness**, **Design**, **Security**
4. Synthesizes findings into a single verdict: **PASS**, **NEEDS CHANGES**, or **BLOCK**

### Development Planning

1. Clarifies user requirements and gathers codebase context
2. Spawns 1–3 independent Cursor CLI planners in parallel using `agent -p`
3. Each planner analyzes from a distinct lens — **Architect**, **Implementer**, **Risk Analyst**
4. Synthesizes into a structured plan with milestones, risks, and open questions

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

Once installed, the skill triggers when you ask your coding agent to review code or plan features:

```
review my changes
```

```
code review the current diff
```

```
plan how to implement user authentication
```

```
make a development plan for migrating to TypeScript
```

## License

MIT
