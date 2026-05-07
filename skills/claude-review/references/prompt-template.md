# Claude Prompt Template: Concept Review

## Initial critique prompt

Use this template for the Step 3 call to Claude. Replace placeholders in angle brackets. Never paste file contents or large text dumps into the prompt — describe the concept and point Claude at the relevant files by absolute path so it can read them itself.

```
You are an adversarial concept reviewer. Review the following concept and identify potential concerns. Your reviewer counterpart (Codex) will challenge your concerns, so cite specific evidence and reasoning.

CONCEPT:
<short inline description of the idea, design, or approach>

CONTEXT FILES: <absolute paths to design docs, RFCs, or code files Claude should read for context, or "none" if absent>

REPO: <absolute path to repo, or "n/a" if the concept is not tied to a specific repo>

If CONTEXT FILES are listed, read them yourself before critiquing. If a REPO is given and the concept refers to current code or a branch/PR, run the appropriate git command (e.g. `git diff origin/main...HEAD`, `gh pr diff <N>`, or read specific files) to ground your critique. Do not ask the user to paste anything.

FOCUS AREAS: <user-specified focus areas, or "none specified" if absent>

Evaluate across these dimensions:
- Feasibility: Can this actually be built as described?
- Scalability: Will this hold up under growth?
- Security: Are there attack vectors or data exposure risks?
- Correctness: Does the logic hold? Are there edge cases that break it?
- Complexity: Is this over-engineered or under-engineered?
- Operations: What are the deployment, monitoring, and maintenance implications?
- Assumptions: What unstated assumptions does this depend on?

For each concern, use this exact format:

CONCERN: <short title>
ASPECT: feasibility | scalability | security | correctness | complexity | operations | assumptions
SEVERITY: critical | warning | info
REASONING: <why this is a problem, with specific references to the concept>
SUGGESTION: <what to do instead>

If you find no significant issues, output:
CONCEPT_LOOKS_SOUND: <brief explanation of why the concept is sound>

Do not add preamble or commentary outside of this format.
```

## Debate challenge prompt

Use this when challenging a Claude concern in the debate loop. If the challenge depends on something in a file, name the file and let Claude re-read it — don't paste the contents.

```
You previously raised this concern about a concept:

CONCERN: <title>
ASPECT: <aspect>
SEVERITY: <severity>
REASONING: <original reasoning>

<Codex's challenge with specific evidence — describe what you saw, point at files by path if needed>

Based on this challenge, respond with exactly one of:
- DEFEND: <maintain your position with additional evidence>
- RETRACT: <withdraw the concern and explain why>
- REVISE: <modify the concern, providing the updated version in full>
```

## Reversed-role challenge prompt

Use when Claude found no issues and Codex is stress-testing. Point Claude back at the same concept and context — don't re-paste the source material.

```
You reviewed the concept (described inline plus context files <list, or "none">) and found no significant issues (CONCEPT_LOOKS_SOUND).

I believe there is a potential concern:

CONCERN: <Codex's concern title>
ASPECT: <aspect>
SEVERITY: <severity>
REASONING: <Codex's reasoning>

Defend the concept against this concern, or concede if it has merit.
Respond with one of:
- DEFEND: <why the concept handles this well>
- CONCEDE: <why this is a real gap>
- PARTIAL: <the concern has some merit but is overstated; here is the remaining gap>
```

## Parsing Claude output

Claude may not follow the format exactly. When parsing:
- Match field names case-insensitively (CONCERN, Concern, concern)
- Strip any preamble text before the first CONCERN: or CONCEPT_LOOKS_SOUND marker
- If a field is missing, infer it from context rather than discarding the whole concern
- If the output is completely unstructured, extract concerns manually and note in the report that formatting was non-standard
