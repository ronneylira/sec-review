# sec-review

Adversarial security code review skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Reviews code changes (PRs, branches, working changes) against 25+ CWE-mapped security anti-patterns and produces a merge/block recommendation.

## Origin

This skill was inspired by and built on top of [Arcanum-Sec/sec-context](https://github.com/Arcanum-Sec/sec-context) by Jason Haddix / Arcanum Information Security. That project provides a curated, LLM-consumable security reference distilled from 150+ sources (CVE databases, academic papers, security blogs, developer forums, GitHub advisories) documenting the dangerous anti-patterns AI coding assistants consistently reproduce. The sec-context repo proposed using this material in LLM system prompts, RAG pipelines, or dedicated security review agents — this skill is an implementation of that idea, packaging the reference material into a change-focused adversarial multi-agent pipeline for Claude Code.

## How it works

```
Reviewer (change analysis) --> Critic (deep verification) --> Lead (merge decision)
```

| Agent | Role | Reference material |
|-------|------|--------------------|
| Reviewer | Analyzes diffs against all 25+ patterns, classifies findings | Full BREADTH reference (~65K tokens) |
| Critic | Challenges findings with attack scenarios, verifies classification | Relevant DEPTH sections only (~15-30K tokens) |
| Lead | Independent judgment, produces merge/block recommendation | No reference material — just both reports |

The pipeline uses **smart context injection**: the orchestrator parses the Reviewer's CWE findings and extracts only the matching deep-dive sections for the Critic.

## What makes this different from sec-hunt

| | sec-hunt | sec-review |
|---|---|---|
| Question | "What vulnerabilities exist in this code?" | "Is this change safe to merge?" |
| Scope | Full file contents | Diffs and changed files |
| Classification | Severity only | **INTRODUCED / WORSENED / PRE-EXISTING** |
| PR awareness | No | Yes — reads title, body, comments |
| Output | Risk rating | **BLOCK / MERGE WITH CONDITIONS / APPROVE** |

## Anti-patterns covered

25+ patterns across 6 categories, each with CWE mapping:

| Category | CWEs | Examples |
|----------|------|----------|
| Secrets & Credentials | CWE-798, CWE-259 | Hardcoded API keys, passwords in config |
| Injection | CWE-89, CWE-78, CWE-90, CWE-643 | SQL, command, LDAP, XPath, NoSQL, SSTI |
| Cross-Site Scripting | CWE-79, CWE-80 | Reflected, stored, DOM-based, mXSS |
| Auth & Sessions | CWE-287, CWE-384, CWE-307 | Weak auth, session fixation, JWT misuse |
| Cryptographic Failures | CWE-327, CWE-330, CWE-326 | Weak algorithms, insecure RNG, ECB mode |
| Input Validation | CWE-20, CWE-22, CWE-434 | Path traversal, file upload, ReDoS |
| + more | CWE-770, CWE-200, CWE-346... | Rate limiting, data exposure, CORS, headers |

## Installation

Copy the `sec-review` folder into your Claude Code skills directory:

```bash
# Clone the repo
git clone https://github.com/ronneylira/sec-review.git

# Copy to Claude Code skills
cp -r sec-review ~/.claude/skills/sec-review
```

### Dependencies

This skill shares reference files with [sec-hunt](https://github.com/ronneylira/sec-hunt). Install sec-hunt first and download the reference files:

```bash
# Install sec-hunt first (contains reference files)
git clone https://github.com/ronneylira/sec-hunt.git
cp -r sec-hunt ~/.claude/skills/sec-hunt

# Download reference files into sec-hunt
cd ~/.claude/skills/sec-hunt
mkdir -p references

curl -sL "https://raw.githubusercontent.com/Arcanum-Sec/sec-context/main/ANTI_PATTERNS_BREADTH.md" \
  -o references/ANTI_PATTERNS_BREADTH.md

curl -sL "https://raw.githubusercontent.com/Arcanum-Sec/sec-context/main/ANTI_PATTERNS_DEPTH.md" \
  -o references/ANTI_PATTERNS_DEPTH.md

# Then install sec-review
cd ~
git clone https://github.com/ronneylira/sec-review.git
cp -r sec-review ~/.claude/skills/sec-review
```

## Usage

```
/sec-review 123                        # Review PR #123
/sec-review -b feature-xyz             # Review branch diff vs main
/sec-review -b feature-xyz --base dev  # Review branch diff vs dev
/sec-review src/                       # Review specific directory
/sec-review src/auth.ts                # Review specific file
/sec-review                            # Review all staged + unstaged changes
```

## Output

The final report includes:

- **Merge decision**: BLOCK / MERGE WITH CONDITIONS / APPROVE
- Confirmed vulnerabilities table with severity, status (INTRODUCED/WORSENED/PRE-EXISTING), CWE, file, line, description, and remediation
- Conditions for merge (if applicable)
- Pre-existing issues (informational, not blocking)
- Low-confidence items flagged for manual review
- Top remediation priorities
- Dismissed findings (collapsed, for transparency)

## Merge decision criteria

- **BLOCK**: Any confirmed INTRODUCED or WORSENED finding rated High or Critical
- **MERGE WITH CONDITIONS**: Only Medium/Low INTRODUCED findings, or High findings that are PRE-EXISTING
- **APPROVE**: No INTRODUCED or WORSENED findings confirmed

## Adversarial scoring

Each agent has an incentive structure that drives accuracy:

- **Reviewer**: Rewarded for finding real vulnerabilities in changed code. Classifies as INTRODUCED/WORSENED/PRE-EXISTING.
- **Critic**: Rewarded for disproving false positives and correcting misclassifications, penalized 3x for wrongly dismissing real vulnerabilities. Can RECLASSIFY findings.
- **Lead**: Scored on correctness. Reads code independently, resolves disagreements, produces the merge decision.

## Tech stack detection

Auto-detects your project's tech stack and focuses agents on relevant language-specific patterns and framework protections.

## Companion skill

[sec-hunt](https://github.com/ronneylira/sec-hunt) — Full codebase security vulnerability hunting. Use for initial audits and periodic full scans.

## Security reference

The anti-pattern reference material is from [Arcanum-Sec/sec-context](https://github.com/Arcanum-Sec/sec-context), a curated LLM-consumable security reference distilled from 150+ sources including CVE databases, academic papers, security blogs, and GitHub advisories. Licensed under CC BY 4.0 by Jason Haddix / Arcanum Information Security.

## License

MIT
