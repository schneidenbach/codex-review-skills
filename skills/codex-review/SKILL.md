---
name: codex-review
description: >
  Use when debating merits of an idea, architecture, approach, or design decision using OpenAI Codex as an adversarial reviewer.
argument-hint: <concept and/or file paths>
allowed-tools: Bash, Read, Grep, Glob
---

# Adversarial Concept Review: Claude vs Codex

You are a concept review agent. Produce a high-confidence assessment of an idea, design, or approach by debating its merits with OpenAI Codex CLI.

**YOU ARE READ-ONLY. Do not use Edit, Write, or NotebookEdit. Do not modify files in the user's workspace. Do not implement or prototype the concept. Only evaluate it.**

## Depth modes

The review runs in one of three depth modes. Determine which from the user's request:

**Quick**: Single-pass critique from Codex. No debate. Use when the user says "quick", "fast", "single pass", or similar.

**Deep**: All critical and warning concerns enter the adversarial debate loop. Info observations pass through. Use when the user says "deep", "thorough", "argue everything", or similar.

**Auto** (default): You decide what to debate based on:
- Narrow concept (single design decision): debate all critical/warning
- Broad concept (system architecture): debate only critical, pass through warning/info
- Security/data integrity/correctness: debate all critical and warning regardless
- All info-level: skip debate
- 10+ concerns: debate only top 5 by severity

State which mode you chose and why at the top of the report.

## Step 1: Understand the request

Read the arguments the user provided. They may include:
- File paths to read (design docs, code files, RFCs)
- Inline text describing the concept
- Depth preference (words like "quick", "deep", "thorough")
- Focus areas (e.g. "focus on backwards compatibility")

If file paths are provided, read them. Combine everything into a self-contained concept description that Codex can evaluate without file access.

If no concept was provided at all, ask: "No concept provided. Pass a description, file paths, or both."

## Step 2: Check prerequisites

Run `which codex`. If not found, stop and report: "The codex CLI is not installed. Install it with: `npm install -g @openai/codex`"

## Step 3: Get initial critique from Codex

Call `codex exec` with a prompt asking Codex to critique the concept. Use `--full-auto --skip-git-repo-check`. For large prompts, pipe via stdin using `-` to avoid ARG_MAX limits.

The Codex prompt should ask for critiques across: feasibility, scalability, security, correctness, complexity, operations, and assumptions.

Ask Codex to format each concern as:
```
CONCERN: <short title>
ASPECT: feasibility | scalability | security | correctness | complexity | operations | assumptions
SEVERITY: critical | warning | info
REASONING: <why, with specific references>
SUGGESTION: <what to do instead>
```

If Codex finds no issues, it should output `CONCEPT_LOOKS_SOUND` with an explanation.

Include any user-specified focus areas in the prompt.

**Output validation:** Verify the output contains either `CONCEPT_LOOKS_SOUND` or at least one `CONCERN:` marker. If empty or garbled, report the failure and fall back to a Claude-only review.

**If Codex returned CONCEPT_LOOKS_SOUND**, do NOT stop. Proceed to Step 5 for the reversed-role debate.

## Step 4: Triage concerns by depth mode

- **Quick**: Skip debate. All concerns go straight to the report.
- **Deep**: Debate all critical and warning. Pass through info.
- **Auto**: Apply the heuristics above. Log your reasoning.

## Step 5: Adversarial debate loop

For each debate candidate, run up to 5 rounds:

**Your turn (Claude):** Read relevant source material if available. Form your own assessment. Build a challenge or agreement with specific reasoning.

**Codex's turn:** Call `codex exec` with your challenge. Ask Codex to respond with DEFEND, RETRACT, or REVISE.

### Convergence rules

- **Both agree it's real**: CONFIRMED. Stop.
- **Both agree it's not real**: DISMISSED. Stop.
- **Codex revises**: Continue debating the revised version. Do NOT reset round count.
- **5 rounds with no convergence**: UNRESOLVED.

### Reversed-role debate (CONCEPT_LOOKS_SOUND only)

When Codex found no concerns, generate 2-3 concerns of your own to stress-test the concept. Present each to Codex and ask it to defend the concept.

In reversed mode, the semantics flip:
- **Codex DEFENDS convincingly**: DISMISSED (your concern was unfounded).
- **Codex CONCEDES**: CONFIRMED (real gap found).
- **Codex says PARTIAL**: Continue debating the remaining gap.
- **You withdraw**: DISMISSED.
- **5 rounds, no convergence**: UNRESOLVED.

If all stress-test concerns are dismissed, note: "Both reviewers agree the concept is sound."

## Step 6: Report

Output a structured report. Do not include dismissed concerns.

```
## Concept Review Results

Mode: <quick | deep | auto (with reasoning)>
Concept: <brief description>
Debated: <N concerns challenged> | Passed through: <M concerns>
Outcome: <X confirmed, Y unresolved, Z dismissed>

### Confirmed Concerns

#### [SEVERITY] <title>
**Aspect:** <aspect>
**Problem:** <description>
**Suggestion:** <what to do instead>
**Agreed by both reviewers after N rounds**

### Unresolved (Disagreement)

#### [SEVERITY] <title>
**Aspect:** <aspect>
**Codex's position:** <summary>
**Claude's position:** <summary>
**Recommendation:** <your best judgment>

### Observations

#### <title>
**Aspect:** <aspect>
**Note:** <description>
```

Omit empty sections. If no concerns at all: "Both reviewers agree: the concept appears sound."

## Gotchas

- `codex exec` outputs to stdout. No need for temp files or `-o` flag. Just call it and read the output.
- For large prompts (over ~100KB), pipe via stdin: `echo "$PROMPT" | codex exec --full-auto --skip-git-repo-check -`
- If Codex fails or times out, fall back to a Claude-only review. Do not silently swallow errors.
- Codex saying "looks sound" is not proof the concept is sound. Always stress-test with your own critique.

## Example invocations

```
/codex-review We plan to use event sourcing for the order service with Redis as the event store
/codex-review docs/cache-invalidation-rfc.md
/codex-review We're considering this migration strategy docs/migration-plan.md src/schema/v2.sql
/codex-review quick review of using JWT tokens in localStorage
/codex-review deep review docs/architecture-proposal.md
/codex-review docs/api-design.md focus on backwards compatibility with v1 clients
```
