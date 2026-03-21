# Codex Review Skills

Adversarial review skills for Claude Code that use OpenAI Codex CLI as a second reviewer.

## Skills

### codex-review
Debate the merits of an idea, architecture, approach, or design decision. Claude and Codex take turns critiquing and defending concerns until they converge on what's real and what's noise.

```
/codex-review We plan to use event sourcing with Redis as the event store
/codex-review docs/architecture-rfc.md
/codex-review deep review of our caching strategy docs/cache.md
```

### codex-code-review
Review code changes for bugs, security issues, and quality problems. Codex flags findings, Claude challenges them, and they debate until each finding is confirmed, dismissed, or marked as unresolved.

```
/codex-code-review
/codex-code-review src/auth.ts src/api/handler.ts
/codex-code-review deep review of branch main
```

## Prerequisites

Requires the [OpenAI Codex CLI](https://github.com/openai/codex):

```bash
npm install -g @openai/codex
```

## Install

### As a Claude Code plugin
```
/plugin install codex-review-skills@codex-review
```

### As personal skills
Copy the skill folders to `~/.claude/skills/`:
```bash
cp -r skills/codex-review ~/.claude/skills/
cp -r skills/codex-code-review ~/.claude/skills/
```

## How it works

Both skills follow the same adversarial pattern:

1. Send code or concept to Codex for initial review
2. Triage findings by severity
3. Claude reads the source material and challenges each finding
4. Codex defends, retracts, or revises
5. Repeat until convergence (max 5 rounds per finding)
6. Produce a structured report with confirmed issues, unresolved disagreements, and observations
