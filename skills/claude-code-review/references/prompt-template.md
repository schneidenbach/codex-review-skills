# Claude Prompt Template: Code Review

## Initial review prompt

Use this template for the Step 4 call to Claude. Replace placeholders in angle brackets.

```
You are an adversarial code reviewer. Review the following code changes and identify potential issues. Your reviewer counterpart (Codex) will challenge your findings, so cite specific evidence.

CODE:
<the diff or file content>

FOCUS AREAS: <user-specified focus areas, or "none specified" if absent>

Look for:
- Bugs: logic errors, off-by-one, null/undefined access, type mismatches
- Security: injection, auth bypass, data exposure, unsafe deserialization
- Race conditions: shared state, missing locks, async ordering issues
- Error handling: swallowed errors, missing cleanup, unclosed resources
- Performance: N+1 queries, unbounded loops, missing pagination, memory leaks

For each finding, use this exact format:

FINDING: <short title>
FILE: <file path>
LINE: <source file line number, NOT the diff offset>
SEVERITY: critical | warning | info
EVIDENCE: <quote the specific code>
REASONING: <why this is a problem>

Important: LINE must be the actual source file line number where the issue occurs, not the diff position or hunk offset.

If you find no issues, output:
NO_ISSUES_FOUND: <brief explanation>

Do not add preamble or commentary outside of this format.
```

## Debate challenge prompt

Use this when challenging a Claude finding in the debate loop.

```
You previously flagged this issue in a code review:

FINDING: <title>
FILE: <file>
LINE: <line>
SEVERITY: <severity>
EVIDENCE: <code quote>
REASONING: <original reasoning>

I read the source file. Here is the actual code with context (lines <start>-<end>):

<code context from source file>

<Codex's challenge with specific evidence>

Based on this challenge, respond with exactly one of:
- DEFEND: <maintain your position with additional evidence>
- RETRACT: <withdraw the finding and explain why>
- REVISE: <modify the finding, providing the updated version in full>
```

## Reversed-role challenge prompt

Use when Claude found no issues and Codex is stress-testing.

```
You reviewed these code changes and found no issues (NO_ISSUES_FOUND).

Here is the diff again:
<diff content>

I believe there is a potential issue:

FINDING: <Codex's finding title>
FILE: <file>
LINE: <line>
SEVERITY: <severity>
EVIDENCE: <code quote>
REASONING: <Codex's reasoning>

Defend the code against this finding, or concede if it has merit.
Respond with one of:
- DEFEND: <why the code is correct>
- CONCEDE: <why this is a real issue>
- PARTIAL: <the finding has some merit but is overstated; here is what remains>
```

## Parsing Claude output

Claude may not follow the format exactly. When parsing:
- Match field names case-insensitively (FINDING, Finding, finding)
- Strip any preamble text before the first FINDING: or NO_ISSUES_FOUND marker
- If a field is missing, infer it from context rather than discarding the whole finding
- If LINE is wrong (doesn't match the EVIDENCE quote), search the source file for the quoted code and correct the line number
- If the output is completely unstructured, extract findings manually and note in the report that formatting was non-standard
