---
name: codebase-diligence
description: >
  Perform exhaustive due diligence on any codebase — examining directory structure,
  file contents, dependency manifests, git history, commit patterns, contributor
  activity, and code quality indicators — then produce a structured, actionable
  markdown report. Use when a user says "audit this codebase", "do a deep dive on
  this repo", "onboard me to this project", "what's the state of this code",
  "review this codebase", "due diligence on this repo", "give me a health check on
  this project", "analyze this codebase", or any time they want a comprehensive
  understanding of an unfamiliar codebase. Also trigger when a user asks about bus
  factor, dependency health, contributor activity, secrets hygiene, or technical
  debt in the context of an existing project. When in doubt, use this skill — a
  structured report prevents missed risks that a casual read-through would overlook.
---

# Codebase Diligence

Analyzes a local git repository across 7 dimensions and writes a structured markdown report to disk.

This is a single-agent, read-only skill. It runs no build tools, modifies no files, and makes no network calls. It uses only standard bash commands (`git log`, `find`, `wc`, `grep`, `cat`) to gather data, then writes one complete markdown report file. The report is the artifact — not the conversation.

---

## Before You Begin

Parse the user's message to extract three things:

1. **Repo path** — look for a file path in the message. If none is given, use the current working directory.
2. **Output path** — default to `<repo-root>/codebase-diligence-report.md`. If the user specified a path, use that.
3. **Focus mode** — scan the message for scope keywords (see the Focus Mode section). If found, note which analysis steps to run.

Verify the repo path exists and is a directory:
```bash
test -d "<repo-path>" && echo "ok" || echo "not found"
```

If the path is not found, stop immediately and tell the user:
> "I couldn't find a directory at `<path>`. Please provide a valid path to a local codebase and try again."

If the path is a file (not a directory):
> "`<path>` appears to be a file, not a directory. Please point me at the root of a codebase."

Otherwise, state the output path in your first response line before doing any analysis:
> "Running a full codebase diligence on `<repo-name>` — I'll write the report to `<output-path>` when I'm done."

If focus mode is active, add:
> "Focusing the analysis on **<focus-area>** — I'll include an Executive Summary and Recommendations alongside the relevant sections."

Then proceed immediately. Do not ask for confirmation.

---

## Step 1: Directory Structure

Emit: `Mapping directory structure...`

Run:
```bash
find <repo> -maxdepth 2 -not -path '*/.git*' -not -path '*/node_modules*' \
  -not -path '*/__pycache__*' -not -path '*/.venv*' \
  -not -path '*/dist*' -not -path '*/build*' \
  -type d | sort
```

From the output:
- Identify project type: **monorepo** (multiple manifests at non-root directories), **single app**, **library**, or **mixed**
- For each major directory, infer its purpose from the name and a spot-check of 2–3 files inside it
- Note any unusual or non-standard directory names

Capture this data for the Repository Overview section of the report.

---

## Step 2: Tech Stack Identification

Emit: `Identifying tech stack from manifests...`

Check for each of these files in order (use `test -f` or `ls`):
- `package.json`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`
- `pyproject.toml`, `requirements.txt`, `poetry.lock`, `Pipfile`
- `Cargo.toml`, `Cargo.lock`
- `go.mod`, `go.sum`
- `pom.xml`, `build.gradle`
- `Gemfile`, `Gemfile.lock`
- `.nvmrc`, `.python-version`, `.tool-versions`
- `Dockerfile`, `docker-compose.yml`

Read each file that exists. Extract: language runtime, framework(s), and major library names.

Report the detected stack in plain English — for example: "Node.js 20, TypeScript, React 18, Express 4, Jest" — not a raw JSON dump.

If no manifest files are found, note this and attempt to infer the stack from file extensions:
```bash
find <repo> -not -path '*/.git*' -not -path '*/node_modules*' \
  \( -name "*.py" -o -name "*.ts" -o -name "*.js" -o -name "*.rb" \
     -o -name "*.go" -o -name "*.rs" -o -name "*.java" \) | head -10
```

Capture manifest file list and stack summary for the Tech Stack section.

---

## Step 3: Dependency Health

Emit: `Checking dependency health and lock files...`

For each detected manifest:
- Count total dependencies (production + development where applicable)
- Note whether versions use pinned values (`1.2.3`) vs. ranges (`^1.2.3`, `~1.2`, `>=1`)
- Check whether a lock file is present

Apply fixed severity thresholds (see Severity Thresholds section).

Note any obviously outdated major versions if detectable by inspection — use only well-known landmarks you can reason about without a network call (e.g., `"react": "^15"` when React 19 is current, `"python": "2.7"`, Django 2.x).

Do NOT call `npm audit`, `pip-audit`, `cargo audit`, or any external registry. Inspection only.

If no manifest files were found in Step 2, skip this section and note in the report: *"No manifest files were detected. Dependency health cannot be assessed."*

---

## Step 4: Git History & Contributor Activity

Emit: `Reading git history...`
Then emit: `Analyzing commit patterns and contributor activity...`

First, check for a `.git` directory:
```bash
test -d "<repo>/.git" && echo "ok" || echo "no git"
```

If absent, skip this step entirely. In the report write: *"No git history found. This directory does not appear to be a git repository, or the `.git` directory is inaccessible."*

If present, run these read-only commands:
```bash
# Total commits
git -C <repo> log --oneline | wc -l

# Commits by month (for frequency trend)
git -C <repo> log --format="%ad" --date=format:"%Y-%m" | sort | uniq -c

# Top contributors by commit count (excluding merges)
git -C <repo> shortlog -sn --no-merges | head -10

# Merge commit count
git -C <repo> log --oneline --merges | wc -l

# First and most recent commit dates
git -C <repo> log --format="%ad" --date=short | tail -1
git -C <repo> log --format="%ad" --date=short | head -1

# Each contributor's most recent commit date
git -C <repo> log --format="%an|%ad" --date=short \
  | sort -t'|' -k1,1 -u | head -20

# Commit message sample (first 20 non-merge messages)
git -C <repo> log --no-merges --format="%s" | head -20
```

Compute from this data:
- **Activity range**: first commit date → most recent commit date
- **Average commits/month**: total commits ÷ months in range
- **Merge ratio**: merge commits ÷ total commits × 100%
- **Bus factor**: if 1 contributor accounts for >80% of commits → 🔴 critical; if 2 contributors together >80% → 🟡 warning
- **Commit convention**: inspect the 20-message sample — look for Conventional Commits prefixes (`feat:`, `fix:`, `chore:`), custom prefixes, ticket numbers, or freeform prose

If `git log` returns empty output (repo initialized but no commits):
> "The repository has been initialized but has no commits yet — git history analysis will be skipped."

---

## Step 5: Code Quality Indicators

Emit: `Checking code quality indicators (tests, lint, CI)...`

**Test directory presence:**
```bash
find <repo> -maxdepth 3 -type d \
  \( -name "test" -o -name "tests" -o -name "spec" \
     -o -name "__tests__" -o -name "test_*" \) \
  -not -path '*/node_modules*' | head -5
```
If found, estimate test-to-source file ratio:
```bash
# Test files (adjust extension for detected language)
find <repo> -not -path '*/node_modules*' -not -path '*/.git*' \
  \( -name "*.test.*" -o -name "*.spec.*" -o -name "test_*.py" \) | wc -l

# Source files
find <repo> -not -path '*/node_modules*' -not -path '*/.git*' \
  -not -path '*/test*' -not -path '*/spec*' \
  \( -name "*.js" -o -name "*.ts" -o -name "*.py" -o -name "*.rb" \
     -o -name "*.go" -o -name "*.rs" \) | wc -l
```

**Linting / formatting config:**
Check for: `.eslintrc*`, `.eslintrc.js`, `.eslintrc.json`, `.prettierrc*`, `ruff.toml`, `.rubocop.yml`, `.flake8`, `pylintrc`, `mypy.ini`, `.stylelint*`, `biome.json`

**CI/CD config:**
Check for: `.github/workflows/*.yml` (count files), `.circleci/config.yml`, `Jenkinsfile`, `.gitlab-ci.yml`, `azure-pipelines.yml`, `.travis.yml`

**Type safety:**
Check for: `tsconfig.json`, `mypy.ini`, `pyrightconfig.json`, `pyright.json`

Report each as ✅ Present or ❌ Absent with a one-line observation.

---

## Step 6: Documentation Assessment

Emit: `Assessing documentation...`

**Standard docs files** — check for each:
- `README.md` — if present: `wc -l README.md`; scan for keywords `install`, `getting started`, `usage`, `quick start` to determine if setup instructions are present
- `CONTRIBUTING.md`
- `CHANGELOG.md`
- `LICENSE` or `LICENSE.md` or `LICENSE.txt`
- `CODEOWNERS`
- `SECURITY.md`

**Inline documentation** — sample 3–5 source files:
```bash
find <repo> -not -path '*/node_modules*' -not -path '*/.git*' \
  \( -name "*.py" -o -name "*.ts" -o -name "*.js" -o -name "*.rb" \
     -o -name "*.go" \) | head -5
```
For each, check for docstrings/JSDoc/rustdoc patterns (`"""`, `/**`, `///`, `#`).

Report each file as ✅ Present or ❌ Absent. For README, note line count and whether setup instructions were detected.

---

## Step 7: Configuration & Secrets Hygiene

Emit: `Scanning for configuration and secrets hygiene...`

**`.gitignore` coverage:**
- Check if `.gitignore` exists
- If present, scan it for these patterns: `.env`, `*.pem`, `*.key`, `secrets.*`, `*.p12`, `credentials*`, `*.pfx`
- Note which common patterns are covered and which are absent

**Potentially committed secret files:**
```bash
find <repo> -not -path '*/.git*' \
  \( -name ".env" -o -name ".env.*" \
     -o -name "*.pem" -o -name "*.key" \
     -o -name "secrets.*" -o -name "credentials*" \) | head -20
```

For each file found, check whether it appears in `.gitignore`. If it does not, flag as 🔴 critical.

Do NOT read the contents of any secret-pattern files — only check for their presence and gitignore status. This is both a security boundary and a context window concern.

---

## Severity Thresholds

Apply these thresholds consistently across all sections. These are fixed — they do not vary by project size or maturity.

**🔴 CRITICAL — always flag:**
- Any secret-pattern file (`.env`, `*.pem`, `*.key`, etc.) found in the repo and not covered by `.gitignore`
- Bus factor: 1 contributor accounts for >80% of commits
- Missing lock file in any project that has a dependency manifest

**🟡 WARNING — flag for all but clearly personal/experimental projects:**
- Bus factor: 2 contributors together account for >80% of commits
- No `.gitignore` file at all
- No CI/CD configuration detected
- No test directory detected
- README exists but contains no setup instructions

**🟢 HEALTHY — note positively when present:**
- Lock file present
- CI/CD configured
- Type safety tooling present
- Multiple contributors with recent activity
- README with setup instructions
- Test directory present with >10% test-to-source file ratio

---

## Focus Mode

Detect focus mode by scanning the user's prompt for these keywords (case-insensitive). If a match is found, run only the listed steps and produce a shorter report with only those sections plus Executive Summary and Recommendations.

| If user mentions... | Mapped focus area | Run steps |
|---|---|---|
| "security" / "secrets" / "hygiene" / "credentials" | Security Hygiene | Step 7 only |
| "git" / "history" / "commits" / "contributors" / "activity" / "bus factor" | Git & Contributors | Step 4 only |
| "dependencies" / "deps" / "packages" / "lock file" | Dependency Health | Steps 2 + 3 |
| "quality" / "tests" / "ci" / "linting" / "test coverage" | Code Quality | Step 5 only |
| "docs" / "documentation" / "readme" / "contributing" | Documentation | Step 6 only |
| "stack" / "tech stack" / "what's this built with" / "language" | Tech Stack | Steps 1 + 2 |

If no keyword matches, run all 7 steps (full analysis).

Before starting a focused analysis, state which sections will be skipped:
> "Focused on **<focus-area>** — skipping [list of omitted sections]."

In the report, omit non-requested sections entirely. Do not write "N/A" or placeholder headings.

---

## Graceful Degradation

Never let a missing piece of data cause the whole analysis to fail. Always write the report with whatever data was gathered, noting gaps explicitly.

| Condition | Behavior |
|---|---|
| No `.git` directory | Skip Step 4; write *"Git history unavailable — no `.git` directory found."* in that section |
| No manifest files found | Note in Steps 2–3; attempt stack inference from file extensions; omit dependency counts |
| Empty repo (no commits) | Note in Step 4; skip contributor and commit analysis |
| Unreadable directory | Note the directory name in the relevant section; continue to next step |
| No write permission at output path | Tell the user: *"I couldn't write the report to `<path>` — permission denied. Try specifying a different output path."* |
| Very large repo (thousands of files) | Use `head -50` on `find` output; note in the report: *"Directory structure is large; summary reflects top two levels only."* |

---

## Synthesize & Write the Report

Emit: `Writing report...`

Gather all findings from the completed steps. Write the report to `<output-path>` in a single operation using this exact structure:

```markdown
# Codebase Diligence Report: [repo-name]
_Generated: [YYYY-MM-DD]_

## Executive Summary
2–4 sentences: what this repo is, its apparent maturity level, and the top 2–3 findings
worth knowing immediately. Close with a count: "X critical issues, Y warnings, Z healthy signals."

## Repository Overview
Project type (monorepo/single app/library/mixed), top-level structure summary, and
a brief description of each major directory's purpose. 1–2 paragraphs.

## Tech Stack
Detected stack in plain English. Manifest files found. Runtime versions if pinned.
If stack was inferred from file extensions rather than manifests, note this.

## Dependency Health
🔴 N critical · 🟡 N warnings · 🟢 N healthy

Per-manifest findings: lock file status (✅/❌), version pinning style, total dependency
count, notable version observations.

## Git History & Contributor Activity
🔴 N critical · 🟡 N warnings · 🟢 N healthy

Total commits, date range, avg commits/month, top contributors with commit counts and
most recent commit date, merge ratio, bus factor assessment, commit convention observed.

## Code Quality Indicators
🔴 N critical · 🟡 N warnings · 🟢 N healthy

- Test directory: ✅/❌ [observation]
- Linting config: ✅/❌ [observation]
- CI/CD: ✅/❌ [observation]
- Type safety: ✅/❌ [observation]

## Documentation Assessment
🔴 N critical · 🟡 N warnings · 🟢 N healthy

- README: ✅/❌ (N lines, setup instructions: yes/no)
- CONTRIBUTING: ✅/❌
- CHANGELOG: ✅/❌
- LICENSE: ✅/❌
- CODEOWNERS: ✅/❌
- Inline docs: observed / not observed

## Configuration & Secrets Hygiene
🔴 N critical · 🟡 N warnings · 🟢 N healthy

- .gitignore: ✅/❌
- Common secret patterns covered: yes/no (list missing patterns if any)
- Committed secret-pattern files: [list files] or "None found"

## Findings Summary
Bulleted list of all 🔴 and 🟡 findings across all sections, with the section each
finding came from. If there are no critical or warning findings, note that here.

## Recommendations
Prioritized list. Each item tagged [CRITICAL], [IMPORTANT], or [NICE TO HAVE].
Each item must name a specific file, pattern, or concrete action — not vague advice.
At least one recommendation per 🔴 finding.
```

After writing the file, deliver an inline summary in chat:

> "Done. Report written to `<output-path>`."
>
> **Key findings:**
> - [3–5 severity-tagged bullets, one per significant finding]
>
> "Full details, section-by-section analysis, and prioritized recommendations are in the report."
