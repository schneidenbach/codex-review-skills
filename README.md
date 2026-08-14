# Adversarial Review Skills

**This repository is retired. Development has moved to [schneidenbach/skills](https://github.com/schneidenbach/skills).**

Adversarial review skills that pair Claude Code and OpenAI Codex against each other. The pair runs in either direction:

- **Claude-side skills** (`codex-review`, `codex-code-review`) run inside Claude Code and call Codex as the second reviewer.
- **Codex-side skills** (`claude-review`, `claude-code-review`) run inside Codex and call Claude as the second reviewer.

Pick the side that matches whichever agent is driving the session.

## Claude-side skills (driven by Claude Code, second opinion from Codex)

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

## Codex-side skills (driven by Codex, second opinion from Claude)

### claude-review
Mirror of `codex-review`, but driven by Codex. Codex critiques the concept, calls `claude -p` for an adversarial second pass, then debates each concern with Claude until convergence.

```
/claude-review We plan to use event sourcing with Redis as the event store
/claude-review docs/architecture-rfc.md
/claude-review deep review of our caching strategy docs/cache.md
```

### claude-code-review
Mirror of `codex-code-review`, but driven by Codex. Codex generates the diff, calls `claude -p` for findings, and debates each finding with Claude.

```
/claude-code-review
/claude-code-review src/auth.ts src/api/handler.ts
/claude-code-review deep review of branch main
```

## Prerequisites

The Claude-side skills need the [OpenAI Codex CLI](https://github.com/openai/codex):

```bash
npm install -g @openai/codex
```

The Codex-side skills need the [Claude Code CLI](https://claude.com/claude-code).

If you want both directions, install both CLIs.

## Install

### Claude-side skills (Claude Code)
```
/plugin marketplace add https://github.com/schneidenbach/codex-review-skills
/plugin install codex-review-skills@codex-review
```

### Codex-side skills (Codex)
Codex loads user-installed skills from `~/.codex/skills/<skill-name>/SKILL.md` — one flat directory per skill at the top level. Inside Codex, ask its built-in `skill-installer` to install from this repo:

> Install skills from `schneidenbach/codex-review-skills`, paths `skills/claude-review` and `skills/claude-code-review`.

Restart Codex afterward to pick up the new skills.

## How it works

All four skills follow the same adversarial pattern:

1. The driver (Claude or Codex) gathers the code or concept
2. The second reviewer (Codex or Claude) returns initial findings or concerns
3. Triage by severity
4. The driver reads the source material and challenges each finding
5. The second reviewer defends, retracts, or revises
6. Repeat until convergence (max 5 rounds per finding)
7. Produce a structured report with confirmed issues, unresolved disagreements, and observations
