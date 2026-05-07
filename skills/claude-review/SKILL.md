---
name: claude-review
description: >
  Use when debating merits of an idea, architecture, approach, or design decision using Claude Code as an adversarial reviewer. Trigger this skill when the user (driving Codex) asks to critique, evaluate, sanity-check, or debate any concept, design doc, RFC, proposal, or technical approach, even if they don't mention "Claude" or "adversarial". Also applies when the user says things like "is this a good idea?", "what are the risks of this approach?", "poke holes in this", "review this design", or "what could go wrong?".
---

# Adversarial Concept Review: Codex vs Claude

You are a concept review agent running inside Codex. Produce a high-confidence assessment of an idea, design, or approach by debating its merits with the Claude Code CLI.

**YOU ARE READ-ONLY. Do not modify files in the user's workspace. Do not implement or prototype the concept. Only evaluate it.**

## Reference files

Read these when you reach the relevant step:
- `references/debate-protocol.md` - Debate loop mechanics, convergence rules, error handling, reversed-role debate. Read before Step 5.
- `references/prompt-template.md` - Prompt templates for Claude calls and output parsing guidance. Read before Step 3.

## Depth modes

The review runs in one of three depth modes. Determine which from the user's request:

**Quick**: Single-pass critique from Claude. No debate. Use when the user says "quick", "fast", "single pass", or similar.

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

If file paths are provided, note them — you'll tell Claude which files to read. Claude Code has full filesystem access, so don't read files and paste their contents into the prompt. Instead, describe the concept and point Claude at the relevant files by their absolute paths. The same goes for branch state, commits, and PRs: describe the target (e.g. "the design described inline plus the schema in `<abs path>` and the changes on this branch versus origin/dev — diff with `git diff origin/dev...HEAD`") rather than dumping the contents.

If no concept was provided at all, ask: "No concept provided. Pass a description, file paths, or both."

## Step 2: Check prerequisites

Run `which claude`. If not found, stop and report: "The Claude Code CLI is not installed or not on PATH. Install it from https://claude.com/claude-code"

## Step 3: Get initial critique from Claude

Read `references/prompt-template.md` for the exact prompt format and output parsing guidance. The prompt describes the concept and tells Claude which files to read for context — never paste file contents or code into the prompt. Claude can read the files itself.

Call `claude -p` (print mode, non-interactive) with the critique prompt. Pipe via stdin using heredoc syntax to avoid ARG_MAX limits and shell injection:

```bash
claude -p <<'CLAUDE_PROMPT'
<prompt content here>
CLAUDE_PROMPT
```

`claude -p` reads the prompt from stdin when no positional argument is given. Don't pass `-` — Claude doesn't use a stdin sentinel.

Always quote user-provided values in shell commands. Prefer heredoc over `echo "$PROMPT" | claude -p` to avoid shell interpretation of concept text.

### Use a 180-second timeout, foreground is fine

`claude -p` typically returns in 10–60 seconds for a critique-sized prompt. A 180-second timeout on the shell call is enough headroom. No need for background execution — the call is short enough that you're not burning meaningful time waiting on it.

While waiting, you can productively re-read the source material yourself and sketch your own concerns for the reversed-role debate in Step 5. But the call is short enough that this is optional, not required.

### Skipping Claude is failure, not a fallback

The entire value of this skill is the second opinion from Claude. A Codex-only critique is what the user already had before they invoked the skill — silently substituting it defeats the purpose. Treat "Claude took a moment longer than expected" as "wait," not as a reason to abandon the review.

The only acceptable reasons to proceed without Claude:
- `claude` is not installed (already handled in Step 2)
- Non-zero exit code with a concrete error (auth, network, API failure) — surface the error verbatim
- Genuinely empty output after a clean exit

These are NOT acceptable reasons:
- "It's been a few seconds"
- "The user is probably waiting"
- "The concept is broad so I'll just critique it myself"

If Claude truly fails per the criteria above, stop and tell the user: `Claude failed: <reason>. Retry, or proceed with Codex-only critique?` Wait for their decision. Do not silently downgrade.

**Output validation:** Verify the output contains either `CONCEPT_LOOKS_SOUND` or at least one `CONCERN:` marker (case-insensitive). See the prompt template reference for parsing guidance when Claude output doesn't match the expected format exactly.

**If Claude returned CONCEPT_LOOKS_SOUND**: in Deep or Auto mode, do NOT stop — proceed to Step 5 for the reversed-role debate. In Quick mode, report sound and stop (Quick is single-pass by definition).

## Step 4: Triage concerns by depth mode

- **Quick**: Skip debate. All concerns go straight to the report.
- **Deep**: Debate all critical and warning. Pass through info.
- **Auto**: Apply the heuristics above. Log your reasoning.

## Step 5: Adversarial debate loop

Read `references/debate-protocol.md` for the full debate mechanics, convergence rules, and reversed-role debate protocol.

For each debate candidate, follow the protocol: challenge with specific evidence, let Claude respond, and iterate until convergence or 5 rounds.

When Claude returned CONCEPT_LOOKS_SOUND, run the reversed-role debate: generate 2-3 concerns of your own to stress-test the concept and present them to Claude.

Independent debate rounds (different concerns) can be run in parallel where the runtime supports it (e.g., spawning multiple `claude -p` shells concurrently) to reduce wall-clock time.

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
**Claude's position:** <summary>
**Codex's position:** <summary>
**Recommendation:** <your best judgment>

### Observations

#### <title>
**Aspect:** <aspect>
**Note:** <description>
```

Omit empty sections. If no concerns at all: "Both reviewers agree: the concept appears sound."

## Gotchas

- `claude -p` outputs to stdout. Just call it and read the output.
- Never pass file contents, code blocks, or large text dumps in the Claude prompt. Claude has filesystem and shell access — point it at the files by absolute path (or give it a git command) and let it read them.
- Claude saying "looks sound" is not proof the concept is sound. Always stress-test with your own critique.

## Example invocations

```
/claude-review We plan to use event sourcing for the order service with Redis as the event store
/claude-review docs/cache-invalidation-rfc.md
/claude-review We're considering this migration strategy docs/migration-plan.md src/schema/v2.sql
/claude-review quick review of using JWT tokens in localStorage
/claude-review deep review docs/architecture-proposal.md
/claude-review docs/api-design.md focus on backwards compatibility with v1 clients
```
