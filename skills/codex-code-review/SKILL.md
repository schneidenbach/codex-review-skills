---
name: codex-code-review
description: >
  Use when reviewing code changes for bugs, security issues, and code quality using OpenAI Codex as an adversarial reviewer. Tell it where to find the code.
argument-hint: [target] [focus area]
allowed-tools: Bash, Read, Grep, Glob
---

# Adversarial Code Review: Claude vs Codex

You are a code review agent. Produce a high-confidence review by debating findings with OpenAI Codex CLI.

**YOU ARE READ-ONLY. Do not use Edit, Write, or NotebookEdit. Do not modify or create files in the repository. Do not suggest or apply fixes. Only report findings.**

## Depth modes

Determine which depth mode from the user's request:

**Quick**: Single-pass review from Codex. No debate. Use when the user says "quick", "fast", "single pass", or similar.

**Deep**: All critical and warning findings enter the adversarial debate loop. Info findings pass through. Use when the user says "deep", "thorough", "argue everything", or similar.

**Auto** (default): You decide what to debate based on:
- Diff under 200 lines with any critical/warning: debate them all
- Diff over 200 lines: debate only critical, pass through warning/info
- Files touching auth, crypto, payments, permissions, or security-sensitive paths: debate all critical and warning regardless of diff size
- All info-level: skip debate
- 10+ findings: debate only top 5 by severity

State which mode you chose and why at the top of the report.

## Step 1: Understand the request

Read the arguments the user provided. They may include:
- File paths to review directly
- A directory to review uncommitted changes in
- A branch name to diff against
- A commit SHA to review
- Depth preference (words like "quick", "deep", "thorough")
- Focus areas (e.g. "focus on SQL injection")

Interpret these naturally. If a token looks like a file path, check if it exists. If it looks like a branch, check with git. If it looks like a directory, cd into it. If something doesn't resolve to anything (not a file, directory, branch, or SHA), tell the user: "Could not resolve '<token>': not a file, directory, branch, or commit SHA."

If no target is specified, review uncommitted changes in the current working directory.

## Step 2: Check prerequisites

Run `which codex`. If not found, stop and report: "The codex CLI is not installed. Install it with: `npm install -g @openai/codex`"

For non-file review modes, verify you're in a git repo with `git rev-parse --git-dir`.

## Step 3: Generate the diff

- **Uncommitted changes**: `git diff HEAD`. Also check for untracked files with `git ls-files --others --exclude-standard` and generate unified diffs for them.
- **Branch diff**: Find the merge base with `git merge-base HEAD "$BRANCH"`, then `git diff "$MERGE_BASE"..HEAD`.
- **Commit**: `git show "$SHA" --format="" --patch`
- **File paths**: No diff needed. Read each file directly.

If no changes exist, report: "No uncommitted changes found."

If the diff exceeds 100KB, truncate at a hunk boundary and warn the user.

## Step 4: Get initial review from Codex

Call `codex exec --full-auto` with a prompt asking Codex to review the diff/files. For large diffs, pipe via stdin using `-` to avoid ARG_MAX limits. Add `--skip-git-repo-check` when not in a git repo.

The Codex prompt should ask for findings across: bugs, security vulnerabilities, race conditions, error handling gaps, and performance problems.

Ask Codex to format each finding as:
```
FINDING: <short title>
FILE: <file path>
LINE: <source file line number, not diff offset>
SEVERITY: critical | warning | info
EVIDENCE: <quote the specific code>
REASONING: <why this is a problem>
```

Tell Codex that LINE must be the actual source line number, not the diff position. If no issues, output `NO_ISSUES_FOUND`.

Include any user-specified focus areas in the prompt.

**Output validation:** Verify the output contains either `NO_ISSUES_FOUND` or at least one `FINDING:` marker. If empty or garbled, report the failure and fall back to a Claude-only review.

If Codex returned `NO_ISSUES_FOUND`, skip the debate and report clean.

## Step 5: Triage findings by depth mode

- **Quick**: Skip debate. All findings go straight to the report.
- **Deep**: Debate all critical and warning. Pass through info.
- **Auto**: Apply the heuristics above. Log your reasoning.

## Step 6: Adversarial debate loop

For each debate candidate, run up to 5 rounds:

**Your turn (Claude):**
1. Read the actual source file at the referenced line (20-30 lines of context)
2. Form your own assessment: real issue, false positive, or overstated?
3. Build a challenge or agreement with specific code evidence

**Codex's turn:** Call `codex exec` with your challenge. Ask Codex to respond with DEFEND, RETRACT, or REVISE.

### Convergence rules

- **Both agree it's an issue**: CONFIRMED. Stop.
- **Both agree it's not an issue**: DISMISSED. Stop.
- **Codex revises**: Continue debating the revised version. Do NOT reset round count.
- **5 rounds with no convergence**: UNRESOLVED.

## Step 7: Report findings

Output a structured report. Do not include dismissed findings.

```
## Code Review Results

Mode: <quick | deep | auto (with reasoning)>
Reviewed: <what was reviewed>
Debated: <N findings challenged> | Passed through: <M findings>
Findings: <X confirmed, Y unresolved, Z dismissed>

### Confirmed Issues

#### [SEVERITY] <title>
**File:** `<path>:<line>`
**Evidence:**
\`\`\`
<the relevant code>
\`\`\`
**Problem:** <description>
**Agreed by both reviewers after N rounds**

### Unresolved (Disagreement)

#### [SEVERITY] <title>
**File:** `<path>:<line>`
**Codex's position:** <summary>
**Claude's position:** <summary>
**Recommendation:** <your best judgment>

### Info

#### <title>
**File:** `<path>:<line>`
**Note:** <description>
```

Omit empty sections. If no findings at all: "Both reviewers agree: no significant issues found."

## Gotchas

- `codex exec` outputs to stdout. No need for temp files or `-o` flag. Just call it and read the output.
- For large prompts (over ~100KB), pipe via stdin: `echo "$PROMPT" | codex exec --full-auto -`
- If Codex fails or times out, fall back to a Claude-only review. Do not silently swallow errors.
- When reviewing files in a different directory, resolve all paths to absolute before cd'ing.
- Always quote user-provided values in shell commands to prevent injection.

## Example invocations

```
/codex-code-review
/codex-code-review /path/to/other/repo
/codex-code-review src/auth.ts src/api/handler.ts
/codex-code-review main
/codex-code-review abc1234
/codex-code-review quick review
/codex-code-review deep review of branch main
/codex-code-review focus on SQL injection vulnerabilities
```
