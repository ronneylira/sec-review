---
name: sec-review
description: "Run adversarial security review on code changes (PRs, branches, paths, working changes). Uses 3 isolated agents (Reviewer, Critic, Lead) with 25+ CWE-mapped anti-pattern references. Produces a merge/block recommendation. Invoke with /sec-review <PR#>, /sec-review -b <branch>, or /sec-review [path]."
argument-hint: "[PR# | -b <branch> [--base <base>] | path]"
disable-model-invocation: true
---

# Sec Review — Adversarial Security Code Review

Run a 3-agent adversarial security review on code changes, powered by 25+ CWE-mapped anti-pattern references. Produces a BLOCK / MERGE WITH CONDITIONS / APPROVE decision.

## Usage

```
/sec-review 123                        # Review PR #123 changed files
/sec-review -b feature-xyz             # Review files changed in feature-xyz vs main
/sec-review -b feature-xyz --base dev  # Review files changed in feature-xyz vs dev
/sec-review src/                       # Review specific directory
/sec-review src/auth.ts                # Review specific file
/sec-review                            # Review all staged + unstaged changes
```

## Target

The raw arguments are: $ARGUMENTS

**Parse the arguments as follows:**

1. If arguments match a number (e.g. `123`): this is a **PR mode**.
   - Run `gh pr diff $NUMBER` using the Bash tool to capture the full diff.
   - Run `gh pr diff $NUMBER --name-only` to get the list of changed files.
   - Run `gh pr view $NUMBER --json title,body,comments,reviews` to capture PR context.
   - If the command fails, report the error and stop.
   - If no files changed, tell the user and stop.
   - The scan target is the list of changed files (full contents) plus the diff.

2. If arguments contain `-b <branch>`: this is a **branch diff mode**.
   - Extract the branch name after `-b`.
   - If `--base <base-branch>` is also present, use that as the base. Otherwise default to `main`.
   - Run `git diff <base>...<branch>` to capture the full diff.
   - Run `git diff --name-only <base>...<branch>` to get the file list.
   - If the command fails, report the error and stop.
   - If no files changed, tell the user and stop.

3. If arguments are a path (file or directory): this is **path mode**.
   - The scan target is the given path.
   - Run `git diff HEAD -- <path>` to capture any diff (may be empty for untracked files).
   - In this mode, treat the review as a full security scan of the path (similar to sec-hunt behavior).

4. If no arguments: this is **working changes mode**.
   - Run `git diff` and `git diff --staged` to capture all diffs.
   - Run `git diff --name-only` and `git diff --name-only --staged` to get file lists.
   - If no files changed, tell the user and stop.

## Execution Steps

You MUST follow these steps in exact order. Each agent runs as a separate subagent via the Agent tool to ensure context isolation.

### Step 0: Detect tech stack

Before launching any agent, detect the project's tech stack by checking for the presence of these files (use Glob):
- `package.json` → Node.js/TypeScript (read it to check for React, Vue, Angular, Next.js, Express, Hono, etc.)
- `requirements.txt` / `pyproject.toml` / `setup.py` → Python
- `go.mod` → Go
- `Cargo.toml` → Rust
- `*.csproj` / `*.sln` → .NET/C#
- `pom.xml` / `build.gradle` → Java
- `Gemfile` → Ruby
- `composer.json` → PHP

Store the detected stack as a short string.

### Step 1: Parse arguments and resolve target

Follow the rules in the **Target** section above. Collect:
- The file list
- The full diff (if available)
- PR context (if PR mode)

### Step 2: Read the prompt files and reference material

Read these files:
- `${CLAUDE_SKILL_DIR}/prompts/reviewer.md`
- `${CLAUDE_SKILL_DIR}/prompts/critic.md`
- `${CLAUDE_SKILL_DIR}/prompts/lead.md`

Read the shared reference from sec-hunt:
- `~/.claude/skills/sec-hunt/references/ANTI_PATTERNS_BREADTH.md`

Do NOT read ANTI_PATTERNS_DEPTH.md yet — it will be read selectively in Step 4.

### Step 3: Run the Reviewer Agent

Launch a general-purpose subagent with the reviewer prompt. Include in the agent's task:
- The file list
- The full diff
- The detected tech stack
- The FULL content of `ANTI_PATTERNS_BREADTH.md` as reference material
- If PR mode: the PR context (title, body, existing comments)

Use `model: "sonnet"` for this agent.

**Important:** Prepend the reviewer prompt, then add:

```
## Your Reference Material (ANTI_PATTERNS_BREADTH)

[full content of ANTI_PATTERNS_BREADTH.md here]

## Tech Stack

[detected tech stack]

## Diff

[full diff here]

## Changed Files

[file list here]

## PR Context (if applicable)

[PR title, body, comments — or "N/A"]
```

Wait for the Reviewer to complete and capture its full output.

### Step 3b: Check for findings

If the Reviewer reported TOTAL FINDINGS: 0, skip Steps 4-5 and go directly to Step 6 with a clean report.

### Step 4: Extract relevant DEPTH sections and run the Critic Agent

Smart context injection — same as sec-hunt:

1. Parse the Reviewer's output to extract the **CWE SUMMARY** line.
2. Read `~/.claude/skills/sec-hunt/references/ANTI_PATTERNS_DEPTH.md`.
3. Map the found CWEs to the 6 deep-dive sections using exact line ranges:
   - CWE-798, CWE-259, CWE-321 → `# Pattern 1: Hardcoded Secrets` (line 129–848)
   - CWE-89, CWE-77, CWE-78 → `# Pattern 2: SQL Injection and Command Injection` (line 849–1659)
   - CWE-79, CWE-80, CWE-83, CWE-87 → `# Pattern 3: Cross-Site Scripting (XSS)` (line 1660–2823)
   - CWE-287, CWE-384, CWE-613, CWE-307, CWE-308, CWE-640, CWE-1275 → `# Pattern 4: Authentication and Session Security` (line 2824–4461)
   - CWE-327, CWE-328, CWE-330, CWE-326, CWE-759 → `# Pattern 5: Cryptographic Failures` (line 4462–5800)
   - CWE-20, CWE-1284, CWE-1333, CWE-22, CWE-180 → `# Pattern 6: Input Validation and Data Sanitization` (line 5801–7194)
   - Note: CWE-1357 (Slopsquatting) has no deep-dive section.
4. Use the Read tool with `offset` and `limit` to extract ONLY the matching sections. If no CWE maps, include the Executive Summary (line 7195–7297) and Testing Recommendations (line 7298–7522).

5. Launch a NEW general-purpose subagent with the critic prompt. Inject:
   - The Reviewer's structured findings (REV-IDs, files, lines, CWEs, claims, evidence, status, points)
   - The extracted DEPTH sections as reference
   - The diff (so the Critic can verify what actually changed)
   - Do NOT include the Reviewer's narrative or methodology text

Use `model: "sonnet"` for this agent.

**Important:** Prepend the critic prompt, then add:

```
## Deep-Dive Reference Material (relevant sections only)

[extracted DEPTH sections here]

## Diff

[full diff here]

## Reviewer Findings to Verify

[Reviewer's structured findings here]
```

Wait for the Critic to complete and capture its full output.

### Step 5: Run the Lead Agent

Launch a NEW general-purpose subagent with the lead prompt. Inject BOTH:
- The Reviewer's full report
- The Critic's full challenge report

Use `model: "sonnet"` for this agent.

Do NOT inject reference material or the diff — the Lead works from both reports and its own code reading.

Wait for the Lead to complete and capture its full output.

### Step 6: Present the Final Report

Display the Lead's final security review report to the user. Include:

1. **Header with context**: tech stack, files changed count, scan mode, PR number (if applicable)
2. The **merge decision** (BLOCK / MERGE WITH CONDITIONS / APPROVE) prominently displayed
3. Summary stats (total found, confirmed, dismissed, by status)
4. Confirmed vulnerabilities table (INTRODUCED/WORSENED first, then PRE-EXISTING)
5. Conditions for merge (if applicable)
6. Low-confidence items flagged for manual review
7. Top remediation priorities
8. A collapsed section with dismissed findings

Format the final output as:

```
## Security Review Results

**Tech stack:** [detected stack]
**Files changed:** [count]
**Review mode:** [PR #N / branch diff / path / working changes]

---

### MERGE DECISION: [BLOCK / MERGE WITH CONDITIONS / APPROVE]

---

[Lead's full report here]

<details>
<summary>Dismissed findings ([count])</summary>

[List of findings dismissed as false positives, with brief reasons]

</details>
```
