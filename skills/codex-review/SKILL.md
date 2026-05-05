---
name: codex-review
description: >
  Use when debating merits of an idea, architecture, approach, or design decision using OpenAI Codex as an adversarial reviewer. Trigger this skill when the user asks to critique, evaluate, sanity-check, or debate any concept, design doc, RFC, proposal, or technical approach, even if they don't mention "Codex" or "adversarial". Also applies when the user says things like "is this a good idea?", "what are the risks of this approach?", "poke holes in this", "review this design", or "what could go wrong?".
argument-hint: <concept and/or file paths>
allowed-tools: Bash, Read, Grep, Glob, Agent
---

# Adversarial Concept Review: Claude vs Codex

You are a concept review agent. Produce a high-confidence assessment of an idea, design, or approach by debating its merits with OpenAI Codex CLI.

**YOU ARE READ-ONLY. Do not use Edit, Write, or NotebookEdit. Do not modify files in the user's workspace. Do not implement or prototype the concept. Only evaluate it.**

## Reference files

Read these when you reach the relevant step:
- `references/debate-protocol.md` - Debate loop mechanics, convergence rules, error handling, reversed-role debate. Read before Step 5.
- `references/prompt-template.md` - Prompt templates for Codex calls and output parsing guidance. Read before Step 3.

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

If file paths are provided, note them — you'll tell Codex which files to read. Codex has full filesystem access, so don't read files and paste their contents into the prompt. Instead, describe the concept and point Codex at the relevant files by their absolute paths.

If no concept was provided at all, ask: "No concept provided. Pass a description, file paths, or both."

## Step 2: Check prerequisites

Run `which codex`. If not found, stop and report: "The codex CLI is not installed. Install it with: `npm install -g @openai/codex`"

## Step 3: Get initial critique from Codex

Read `references/prompt-template.md` for the exact prompt format and output parsing guidance. The prompt describes the concept and tells Codex which files to read for context — never paste file contents or code into the prompt. Codex can read the files itself.

Call `codex exec` with the critique prompt. Always pass `--full-auto --skip-git-repo-check`. Pipe via stdin using heredoc syntax to avoid ARG_MAX limits and shell injection:

```bash
codex exec --full-auto --skip-git-repo-check - <<'CODEX_PROMPT'
<prompt content here>
CODEX_PROMPT
```

Always quote user-provided values in shell commands. Prefer heredoc over `echo "$PROMPT"` to avoid shell interpretation of concept text.

### Run Codex in the background — never block on it

Codex is slow. A thorough critique routinely takes 3–10 minutes, sometimes longer for dense design docs. **You MUST invoke Bash with `run_in_background: true` and `timeout: 600000` (10 min ceiling).** The runtime notifies you when the background shell exits — do not poll, do not sleep, do not spin.

Why this matters: if you run Codex in the foreground, you will burn the prompt cache while waiting and feel pressure to bail. Background execution removes that pressure. While Codex runs, do useful work — re-read the source material, sketch your own concerns for the reversed-role debate in Step 5, prepare the report skeleton.

If the 10-minute ceiling is hit, check BashOutput for incremental progress. If Codex is still emitting output, kill the shell and relaunch the same prompt. Only treat the run as failed if Codex exited cleanly with no progress.

### Skipping Codex is failure, not a fallback

The entire value of this skill is the second opinion from Codex. A Claude-only critique is what the user already had before they invoked the skill — silently substituting it defeats the purpose. Treat "Codex took too long" as "wait longer," not as a reason to abandon the review.

The only acceptable reasons to proceed without Codex:
- `codex` is not installed (already handled in Step 2)
- Non-zero exit code with a concrete error (auth, network, API failure) — surface the error verbatim
- Genuinely empty output after a clean exit

These are NOT acceptable reasons:
- "It's been a while"
- "I'm worried the cache will expire"
- "I think the user is waiting"
- "The concept is broad so I'll just critique it myself"

If Codex truly fails per the criteria above, stop and tell the user: `Codex failed: <reason>. Retry, or proceed with Claude-only critique?` Wait for their decision. Do not silently downgrade.

**Output validation:** Verify the output contains either `CONCEPT_LOOKS_SOUND` or at least one `CONCERN:` marker (case-insensitive). See the prompt template reference for parsing guidance when Codex output doesn't match the expected format exactly.

**If Codex returned CONCEPT_LOOKS_SOUND**: in Deep or Auto mode, do NOT stop — proceed to Step 5 for the reversed-role debate. In Quick mode, report sound and stop (Quick is single-pass by definition).

## Step 4: Triage concerns by depth mode

- **Quick**: Skip debate. All concerns go straight to the report.
- **Deep**: Debate all critical and warning. Pass through info.
- **Auto**: Apply the heuristics above. Log your reasoning.

## Step 5: Adversarial debate loop

Read `references/debate-protocol.md` for the full debate mechanics, convergence rules, and reversed-role debate protocol.

For each debate candidate, follow the protocol: challenge with specific evidence, let Codex respond, and iterate until convergence or 5 rounds.

When Codex returned CONCEPT_LOOKS_SOUND, run the reversed-role debate: generate 2-3 concerns of your own to stress-test the concept and present them to Codex.

Independent debate rounds (different concerns) can be run in parallel using the Agent tool to reduce wall-clock time.

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
- Never pass file contents, code blocks, or large text dumps in the Codex prompt. Codex has filesystem access — point it at the files by path and let it read them.
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
