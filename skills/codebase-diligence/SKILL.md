---
name: codebase-diligence
description: >
  Perform exhaustive due diligence on any codebase — examining directory structure,
  tech stack, AI tool usage, dependencies, git history, code quality, security posture,
  documentation, architecture, infrastructure, licensing, API surface, and team process
  maturity — then produce a structured, executive-ready markdown report with a scorecard,
  severity-tagged findings, and drill-down detail sections. Supports three depth tiers:
  Quick Scan (fast overview), Standard (comprehensive single-agent), and Deep Audit
  (multi-agent with specialist analysts for AI tools, security, and infrastructure).
  Use when a user says "audit this codebase", "do a deep dive on this repo", "onboard me
  to this project", "what's the state of this code", "review this codebase", "due diligence
  on this repo", "give me a health check", "analyze this codebase", "M&A due diligence",
  "vendor assessment", or any time they want a comprehensive understanding of an unfamiliar
  codebase. Also trigger when a user asks about AI tool usage, bus factor, dependency health,
  contributor activity, secrets hygiene, security posture, or technical debt in the context
  of an existing project.
---

# Codebase Diligence

Analyzes a local git repository across up to 13 dimensions and writes a structured, executive-ready markdown report to disk. Reports are designed for executive readers — scorecard at the top, severity-tagged findings, and anchor links to drill into detail.

This is a read-only skill. It runs no build tools, modifies no files (except the report), and makes no network calls. It uses only standard bash commands (`git log`, `find`, `wc`, `grep`, `cat`) to gather data.

## Depth Tiers

| Tier | Trigger Keywords | Approach | Coverage |
|------|-----------------|----------|----------|
| **Quick Scan** | "quick", "fast", "glance", "overview", "skim", "brief", "snapshot" | Single-agent, 5 lightweight steps | Structure, stack, AI check, git summary, secrets |
| **Standard** | Default (no depth keyword) | Single-agent, 13 steps | All 13 analysis areas |
| **Deep Audit** | "deep", "thorough", "comprehensive", "full audit", "due diligence", "M&A", "acquisition", "vendor assessment" | Multi-agent (3 specialists + orchestrator) | All 13 areas with specialist depth in AI tools, security, and infrastructure |

---

## Before You Begin

Parse the user's message to extract four things:

1. **Repo path** — look for a file path in the message. If none is given, use the current working directory.
2. **Output path** — default to `<repo-root>/codebase-diligence-report.md`. If the user specified a path, use that.
3. **Depth tier** — scan for depth keywords (case-insensitive). If none match, default to Standard.
4. **Focus mode** — scan for scope keywords (see Focus Mode section). Focus mode and depth tiers are orthogonal — you can run a Deep Audit focused on AI tools.

Verify the repo path exists and is a directory:
```bash
test -d "<repo-path>" && echo "ok" || echo "not found"
```

If the path is not found, stop immediately:
> "I couldn't find a directory at `<path>`. Please provide a valid path to a local codebase and try again."

If the path is a file:
> "`<path>` appears to be a file, not a directory. Please point me at the root of a codebase."

Otherwise, state the depth tier and output path:
> "Running a **[Quick Scan / Standard Analysis / Deep Audit]** on `<repo-name>` — report will be written to `<output-path>`."

If focus mode is active, add:
> "Focusing on **<focus-area>** — other sections will be omitted."

Then proceed immediately. Do not ask for confirmation.

---

# Standard Analysis (Default Tier)

Run all 13 steps sequentially. This is the foundation — Quick Scan runs a subset, Deep Audit augments with specialist agents.

---

## Step 1: Directory Structure

Emit: `Mapping directory structure...`

```bash
find <repo> -maxdepth 2 -not -path '*/.git*' -not -path '*/node_modules*' \
  -not -path '*/__pycache__*' -not -path '*/.venv*' \
  -not -path '*/dist*' -not -path '*/build*' \
  -type d | sort | head -80
```

From the output:
- Identify project type: **monorepo** (multiple manifests at non-root directories), **single app**, **library**, or **mixed**
- For each major directory, infer its purpose from the name and a spot-check of 2-3 files inside it
- Note any unusual or non-standard directory names

---

## Step 2: Tech Stack Identification

Emit: `Identifying tech stack...`

Check for these manifest files (use `test -f` or `ls`):
- `package.json`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`
- `pyproject.toml`, `requirements.txt`, `poetry.lock`, `Pipfile`
- `Cargo.toml`, `Cargo.lock`
- `go.mod`, `go.sum`
- `pom.xml`, `build.gradle`, `build.gradle.kts`
- `Gemfile`, `Gemfile.lock`
- `.nvmrc`, `.python-version`, `.tool-versions`
- `Dockerfile`, `docker-compose.yml`, `docker-compose.yaml`

Read each file that exists. Extract: language runtime, framework(s), and major library names. Report the stack in plain English (e.g., "Node.js 20, TypeScript, React 18, Express 4, Jest").

If no manifest files found, infer from file extensions:
```bash
find <repo> -not -path '*/.git*' -not -path '*/node_modules*' \
  \( -name "*.py" -o -name "*.ts" -o -name "*.js" -o -name "*.rb" \
     -o -name "*.go" -o -name "*.rs" -o -name "*.java" \) | head -10
```

---

## Step 3: AI Tools & LLM Usage

**This is the primary focus section.** Load `references/ai-detection-patterns.md` for the full detection battery.

Emit: `Analyzing AI tool usage and LLM integrations...`

### 3a: AI Coding Assistant Configs

Check for all config files/directories listed in Section 1 of the reference file. For each found, note the tool name and files present.

### 3b: AI-Generated Code Markers

Run the git commit pattern searches from Section 2 of the reference file:
- Count commits with AI co-author trailers
- Count commits with AI-related messages
- Calculate AI co-author ratio: AI-marked commits / total commits

### 3c: LLM SDK Dependencies

Check all manifest files against the package lists in Section 3 of the reference file. Note which packages are production vs dev dependencies where distinguishable.

### 3d: AI Environment Variables

Run the env var search from Section 4 of the reference file. Check `.env.example` files specifically — these indicate expected AI service connections.

### 3e: Vector Store & RAG Configurations

Check for patterns from Section 5 of the reference file.

### 3f: AI Governance

Check for governance documents from Section 7 of the reference file. Read `CONTRIBUTING.md` if it exists and scan for AI-related guidance.

### 3g: Score the Four Dimensions

Apply the scoring rubrics from Section 8 of the reference file:
- **AI Tool Adoption Level**: None / Exploratory / Integrated / AI-Native
- **AI Dependency Risk**: Low / Medium / High
- **Code Provenance Clarity**: Clear / Partial / Opaque
- **AI Governance Maturity**: None / Informal / Documented / Enforced

---

## Step 4: Dependency Health

Emit: `Checking dependency health...`

For each detected manifest:
- Count total dependencies (production + development)
- Note version pinning style: exact (`1.2.3`) vs. ranges (`^1.2.3`, `~1.2`, `>=1`)
- Check lock file presence
- Note obviously outdated major versions detectable by inspection (e.g., `"react": "^15"`, `"python": "2.7"`, Django 2.x)

Do NOT run `npm audit`, `pip-audit`, or any external audit tool.

If no manifests found: *"No manifest files detected. Dependency health cannot be assessed."*

---

## Step 5: Git History & Contributor Activity

Emit: `Analyzing git history and contributors...`

Check for `.git` directory. If absent, skip with note.

If present, run:
```bash
# Total commits
git -C <repo> log --oneline | wc -l

# Commits by month
git -C <repo> log --format="%ad" --date=format:"%Y-%m" | sort | uniq -c

# Top contributors (excluding merges)
git -C <repo> shortlog -sn --no-merges | head -10

# Merge commit count
git -C <repo> log --oneline --merges | wc -l

# First and most recent commit dates
git -C <repo> log --format="%ad" --date=short | tail -1
git -C <repo> log --format="%ad" --date=short | head -1

# Each contributor's most recent commit
git -C <repo> log --format="%an|%ad" --date=short | sort -t'|' -k1,1 -u | head -20

# Commit message sample
git -C <repo> log --no-merges --format="%s" | head -20
```

Compute:
- **Activity range**: first → most recent commit
- **Average commits/month**: total ÷ months in range
- **Merge ratio**: merge commits ÷ total × 100%
- **Bus factor**: 1 contributor >80% → 🔴; 2 contributors >80% → 🟡
- **Commit convention**: Conventional Commits, custom prefixes, ticket numbers, or freeform

---

## Step 6: Code Quality Indicators

Emit: `Checking code quality indicators...`

**Test directory presence:**
```bash
find <repo> -maxdepth 3 -type d \
  \( -name "test" -o -name "tests" -o -name "spec" \
     -o -name "__tests__" -o -name "test_*" \) \
  -not -path '*/node_modules*' | head -5
```

If found, estimate test-to-source ratio:
```bash
find <repo> -not -path '*/node_modules*' -not -path '*/.git*' \
  \( -name "*.test.*" -o -name "*.spec.*" -o -name "test_*.py" \) | wc -l

find <repo> -not -path '*/node_modules*' -not -path '*/.git*' \
  -not -path '*/test*' -not -path '*/spec*' \
  \( -name "*.js" -o -name "*.ts" -o -name "*.py" -o -name "*.rb" \
     -o -name "*.go" -o -name "*.rs" \) | wc -l
```

**Linting / formatting:** Check for `.eslintrc*`, `.prettierrc*`, `ruff.toml`, `.rubocop.yml`, `.flake8`, `pylintrc`, `mypy.ini`, `.stylelint*`, `biome.json`, `dprint.json`

**CI/CD:** Check for `.github/workflows/*.yml` (count), `.circleci/config.yml`, `Jenkinsfile`, `.gitlab-ci.yml`, `azure-pipelines.yml`, `.travis.yml`

**Type safety:** Check for `tsconfig.json`, `mypy.ini`, `pyrightconfig.json`, `pyright.json`

Report each as ✅ Present or ❌ Absent with one-line observation.

---

## Step 7: Security Posture

Emit: `Evaluating security posture...`

### Secrets Hygiene

- Check `.gitignore` existence and coverage of: `.env`, `*.pem`, `*.key`, `secrets.*`, `*.p12`, `credentials*`, `*.pfx`
- Scan for committed secret-pattern files:
```bash
find <repo> -not -path '*/.git*' \
  \( -name ".env" -o -name ".env.*" -o -name "*.pem" -o -name "*.key" \
     -o -name "secrets.*" -o -name "credentials*" \) | head -20
```
- For each found, check gitignore coverage. Flag uncovered files as 🔴 critical.
- Do NOT read secret file contents.

### Authentication Patterns

Scan for common auth indicators:
```bash
grep -rl --include="*.{ts,js,py,rb,go,java}" \
  -iE "(jwt|oauth|passport|auth0|firebase.auth|session|bcrypt|argon2)" \
  <repo> | head -10
```
Note which patterns are detected. If none found, note absence.

### Input Validation & Injection Risk

```bash
# SQL string concatenation (potential injection)
grep -rn --include="*.{ts,js,py,rb,go,java}" \
  -E "(execute\(.*\+|query\(.*\+|f\".*SELECT|f\".*INSERT|f\".*UPDATE)" \
  <repo> | head -10

# Check for rate limiting
grep -rl --include="*.{ts,js,py,rb,go,java,yml,yaml}" \
  -iE "(rate.?limit|throttle|express-rate-limit|slowapi|rack-throttle)" \
  <repo> | head -5
```

---

## Step 8: Documentation Assessment

Emit: `Assessing documentation...`

Check for standard docs:
- `README.md` — if present: `wc -l`; scan for `install`, `getting started`, `usage`, `quick start`
- `CONTRIBUTING.md`
- `CHANGELOG.md`
- `LICENSE` / `LICENSE.md` / `LICENSE.txt`
- `CODEOWNERS`
- `SECURITY.md`
- `docs/` directory presence and file count

Sample 3-5 source files for inline docs (docstrings, JSDoc, rustdoc patterns).

Report each as ✅ Present or ❌ Absent. For README, note line count and setup instruction presence.

---

## Step 9: Architecture & Complexity

Emit: `Analyzing architecture and complexity...`

### Code Size

```bash
# Lines of code by language (top extensions)
find <repo> -not -path '*/.git*' -not -path '*/node_modules*' \
  -not -path '*/vendor*' -not -path '*/dist*' \
  \( -name "*.ts" -o -name "*.js" -o -name "*.py" -o -name "*.go" \
     -o -name "*.rs" -o -name "*.java" -o -name "*.rb" \) \
  -exec wc -l {} + 2>/dev/null | tail -1

# Largest files
find <repo> -not -path '*/.git*' -not -path '*/node_modules*' \
  -not -path '*/vendor*' -not -path '*/dist*' -not -path '*/*.lock' \
  -type f -name "*.{ts,js,py,go,rs,java,rb}" \
  -exec wc -l {} + 2>/dev/null | sort -rn | head -10
```

### Entry Points

Look for main files, API route registrations, CLI entrypoints:
```bash
find <repo> -maxdepth 3 -not -path '*/.git*' -not -path '*/node_modules*' \
  \( -name "main.*" -o -name "index.*" -o -name "app.*" -o -name "server.*" \
     -o -name "cli.*" -o -name "__main__.py" -o -name "manage.py" \) | head -10
```

### Coupling Indicators

```bash
# Cross-directory imports (sample)
grep -rn --include="*.{ts,js}" "from.*\.\./\.\." <repo> | wc -l
grep -rn --include="*.py" "from.*\.\." <repo> | wc -l
```

Note: high cross-directory import counts suggest tight coupling. Report the count with context.

---

## Step 10: Infrastructure & Deployment

Emit: `Checking infrastructure and deployment...`

**Cloud provider indicators:**
```bash
find <repo> -not -path '*/.git*' -not -path '*/node_modules*' \
  \( -name "*.tf" -o -name "*.tfvars" -o -name "cdk.json" \
     -o -name "template.yaml" -o -name "serverless.yml" \
     -o -name "pulumi.*" -o -name "cloudformation*" \) | head -10
```

**Containers:**
- `Dockerfile` (count), `docker-compose.yml`, `.dockerignore`
- Kubernetes: `k8s/`, `kubernetes/`, `helm/`, files matching `*.yaml` with `apiVersion:` + `kind:`

**Monitoring / Observability:**
```bash
grep -rl --include="*.{ts,js,py,rb,go,java,yml,yaml,json}" \
  -iE "(datadog|newrelic|sentry|prometheus|grafana|opentelemetry|otel|pagerduty|honeycomb)" \
  <repo> | head -10
```

**Environment management:** Count `.env*` files, check for environment-specific configs (e.g., `config/production.*`, `config/staging.*`).

---

## Step 11: Licensing & Compliance

Emit: `Checking licensing and compliance...`

- Read `LICENSE` file if present — identify the license type (MIT, Apache 2.0, GPL, BSD, proprietary, etc.)
- Check manifests for `license` field
- Scan dependency manifests for license declarations if available (e.g., `package.json` "license" fields in lock files)
- Flag potential copyleft contamination: any GPL-family dependency in a non-GPL project
- Check for SBOM files: `sbom.json`, `sbom.spdx`, `bom.xml`, `cyclonedx.json`

---

## Step 12: API Surface & Integrations

Emit: `Mapping API surface and integrations...`

**API specs:**
```bash
find <repo> -not -path '*/.git*' -not -path '*/node_modules*' \
  \( -name "openapi.*" -o -name "swagger.*" -o -name "*.graphql" \
     -o -name "*.gql" -o -name "*.proto" -o -name "api-spec*" \) | head -10
```

**Third-party service integrations:** Check manifests and source for:
- Payment: `stripe`, `braintree`, `square`, `paypal`
- Communication: `twilio`, `sendgrid`, `@sendgrid`, `mailgun`, `postmark`
- Storage: `aws-sdk`, `@aws-sdk`, `boto3`, `google-cloud-storage`, `@google-cloud/storage`
- Auth services: `auth0`, `firebase`, `clerk`, `supabase`
- Analytics: `segment`, `mixpanel`, `amplitude`, `posthog`

**Webhook handlers:** Search for `webhook` in route files and configs.

---

## Step 13: Team & Process Maturity

Emit: `Assessing team and process maturity...`

- **PR template:** `.github/pull_request_template.md` or `.github/PULL_REQUEST_TEMPLATE/`
- **Issue templates:** `.github/ISSUE_TEMPLATE/` — count templates
- **Release process:** `git -C <repo> tag | tail -10` — check for semver tags; check for release branches
- **Branch strategy:** Count branches matching `release/*`, `develop`, `staging`, `main`/`master`
- **Commit signing:** `git -C <repo> log --format="%G?" | head -20 | sort | uniq -c` (G=good, N=none)
- **Code review indicators:** Merge ratio (from Step 5) as proxy for PR-based workflow

---

## Severity Thresholds

Apply consistently across all sections. Fixed — do not vary by project size.

### 🔴 CRITICAL — always flag:
- Secret-pattern file found in repo and not in `.gitignore`
- Bus factor: 1 contributor >80% of commits
- Missing lock file with dependency manifest present
- AI API keys found in committed files
- LLM API calls in critical paths with no fallback (Deep Audit only)
- No AI governance with AI-Native adoption level

### 🟡 WARNING — flag for all but clearly personal projects:
- Bus factor: 2 contributors >80% of commits
- No `.gitignore` file
- No CI/CD configuration
- No test directory
- README without setup instructions
- AI tool configs present but no governance documentation
- AI co-author ratio >50% with no provenance policy
- LLM dependencies without version pinning
- SQL string concatenation patterns detected
- No authentication patterns in a web application
- GPL dependency in a non-GPL project

### 🟢 HEALTHY — note positively:
- Lock file present
- CI/CD configured
- Type safety tooling present
- Multiple contributors with recent activity
- README with setup instructions
- Test directory with >10% test-to-source ratio
- AI governance documentation present
- LLM dependencies pinned with lock file
- OpenAPI/GraphQL spec present
- Monitoring/observability tools configured
- PR template present
- Semver release tags

---

## Focus Mode

Detect focus keywords (case-insensitive). Run only listed steps + Executive Summary and Recommendations. Focus mode works with any depth tier.

| User mentions... | Focus area | Steps |
|---|---|---|
| "ai" / "AI tools" / "copilot" / "LLM" / "AI usage" | AI Tools & LLM Usage | 2 + 3 |
| "security" / "secrets" / "hygiene" / "credentials" / "auth" | Security Posture | 7 |
| "git" / "history" / "commits" / "contributors" / "bus factor" | Git & Contributors | 5 |
| "dependencies" / "deps" / "packages" / "lock file" | Dependency Health | 2 + 4 |
| "quality" / "tests" / "ci" / "linting" | Code Quality | 6 |
| "docs" / "documentation" / "readme" | Documentation | 8 |
| "stack" / "tech stack" / "built with" / "language" | Tech Stack | 1 + 2 |
| "architecture" / "complexity" / "coupling" | Architecture | 1 + 9 |
| "infrastructure" / "deploy" / "cloud" / "k8s" | Infrastructure | 10 |
| "license" / "compliance" / "SBOM" | Licensing | 11 |
| "api" / "integrations" / "endpoints" / "graphql" | API Surface | 12 |
| "process" / "maturity" / "team health" / "releases" | Team Maturity | 5 + 13 |

If no keyword matches, run all steps.

Before starting a focused analysis:
> "Focused on **<focus-area>** — skipping [list of omitted sections]."

Omit non-requested sections entirely in the report.

---

## Graceful Degradation

Never let missing data fail the analysis. Write the report with gathered data, noting gaps.

| Condition | Behavior |
|---|---|
| No `.git` directory | Skip Step 5; note git history unavailable |
| No manifest files | Note in Steps 2/4; infer stack from extensions |
| Empty repo (no commits) | Skip contributor/commit analysis |
| Unreadable directory | Note and continue |
| No write permission | Tell user and suggest alternative path |
| Very large repo | Use `head -80` on `find`; note in report |

---

## Synthesize & Write the Report

Emit: `Synthesizing findings and writing report...`

Gather all findings. Write the report to `<output-path>` using this structure:

```markdown
# Codebase Diligence Report: [repo-name]
_Generated: [YYYY-MM-DD] | Depth: [Quick Scan / Standard / Deep Audit]_

## Executive Summary

[3-5 sentences: what this repo is, its maturity level, top 2-3 findings.
 Close with: "X critical findings, Y warnings, Z healthy signals identified
 across N analysis areas."]

### Scorecard

| Category | Rating | 🔴 | 🟡 | 🟢 | Details |
|----------|--------|-----|-----|-----|---------|
| AI Tools & LLM Usage | [🔴/🟡/🟢] | N | N | N | [→ Detail](#ai-tools--llm-usage) |
| Dependency Health | [🔴/🟡/🟢] | N | N | N | [→ Detail](#dependency-health) |
| Git History & Contributors | [🔴/🟡/🟢] | N | N | N | [→ Detail](#git-history--contributor-activity) |
| Code Quality | [🔴/🟡/🟢] | N | N | N | [→ Detail](#code-quality-indicators) |
| Security Posture | [🔴/🟡/🟢] | N | N | N | [→ Detail](#security-posture) |
| Documentation | [🔴/🟡/🟢] | N | N | N | [→ Detail](#documentation-assessment) |
| Architecture & Complexity | [🔴/🟡/🟢] | N | N | N | [→ Detail](#architecture--complexity) |
| Infrastructure & Deployment | [🔴/🟡/🟢] | N | N | N | [→ Detail](#infrastructure--deployment) |
| Licensing & Compliance | [🔴/🟡/🟢] | N | N | N | [→ Detail](#licensing--compliance) |
| API Surface & Integrations | [🔴/🟡/🟢] | N | N | N | [→ Detail](#api-surface--integrations) |
| Team & Process Maturity | [🔴/🟡/🟢] | N | N | N | [→ Detail](#team--process-maturity) |

[Category Rating = worst severity in that section: any 🔴 → 🔴, else any 🟡 → 🟡, else 🟢]

### Top Risks

1. 🔴 [Risk description] — [→ Section](#anchor)
2. 🟡 [Risk description] — [→ Section](#anchor)
[... up to 5 risks, ordered by severity then impact]

---

## AI Tools & LLM Usage
**[🔴/🟡/🟢] | 🔴 N · 🟡 N · 🟢 N**

### AI Coding Assistants Detected
[Table or list of tools found with file paths as evidence]
[If none: "No AI coding assistant configurations detected."]

### LLM API Integrations
[Dependencies found, which manifests, production vs dev]
[If none: "No LLM SDK dependencies found in project manifests."]

### AI-Generated Code Indicators
[Co-author trailer count and ratio, AI-referencing commit messages]
[If none: "No AI-generated code markers found in git history."]

### AI Environment Variables
[Env vars referenced, in which files]

### AI Governance Assessment
[Governance docs found or absent; scoring on 4 dimensions]

**AI Maturity Scorecard:**
| Dimension | Rating |
|-----------|--------|
| Adoption Level | None / Exploratory / Integrated / AI-Native |
| Dependency Risk | Low / Medium / High |
| Code Provenance Clarity | Clear / Partial / Opaque |
| Governance Maturity | None / Informal / Documented / Enforced |

---

## Repository Overview
[Project type, top-level structure, major directory purposes. 1-2 paragraphs.]

## Tech Stack
[Stack in plain English. Manifests found. Runtime versions if pinned.]

## Dependency Health
**[🔴/🟡/🟢] | 🔴 N · 🟡 N · 🟢 N**
[Per-manifest: lock status, pinning style, count, version notes]

## Git History & Contributor Activity
**[🔴/🟡/🟢] | 🔴 N · 🟡 N · 🟢 N**
[Total commits, date range, avg/month, top contributors, bus factor, conventions]

## Code Quality Indicators
**[🔴/🟡/🟢] | 🔴 N · 🟡 N · 🟢 N**
- Test directory: ✅/❌ [observation]
- Linting config: ✅/❌ [observation]
- CI/CD: ✅/❌ [observation]
- Type safety: ✅/❌ [observation]

## Security Posture
**[🔴/🟡/🟢] | 🔴 N · 🟡 N · 🟢 N**
- Secrets hygiene: [findings]
- Authentication: [patterns detected or absent]
- Input validation: [injection risk indicators]
- Rate limiting: [detected or absent]

## Documentation Assessment
**[🔴/🟡/🟢] | 🔴 N · 🟡 N · 🟢 N**
[Standard docs checklist, README details, inline docs]

## Architecture & Complexity
**[🔴/🟡/🟢] | 🔴 N · 🟡 N · 🟢 N**
[Code size, largest files, entry points, coupling indicators]

## Infrastructure & Deployment
**[🔴/🟡/🟢] | 🔴 N · 🟡 N · 🟢 N**
[Cloud provider, containers, K8s, monitoring, environment management]

## Licensing & Compliance
**[🔴/🟡/🟢] | 🔴 N · 🟡 N · 🟢 N**
[License type, dependency licenses, copyleft risk, SBOM]

## API Surface & Integrations
**[🔴/🟡/🟢] | 🔴 N · 🟡 N · 🟢 N**
[API specs, third-party services, webhooks]

## Team & Process Maturity
**[🔴/🟡/🟢] | 🔴 N · 🟡 N · 🟢 N**
[PR/issue templates, releases, branch strategy, commit signing, review indicators]

---

## Findings Summary

All 🔴 and 🟡 findings, bulleted, with section attribution:
- 🔴 [Finding] — [Section]
- 🟡 [Finding] — [Section]

If none: "No critical or warning findings identified."

## Recommendations

Prioritized list. Each tagged [CRITICAL], [IMPORTANT], or [NICE TO HAVE].
Each must name a specific file, pattern, or concrete action.
At least one recommendation per 🔴 finding.
```

After writing the file, deliver an inline summary:

> "Done. Report written to `<output-path>`."
>
> **Key findings:**
> - [3-5 severity-tagged bullets]
>
> "Full details and recommendations are in the report."

---

# Quick Scan Tier

When depth keywords match Quick Scan, run an abbreviated analysis.

**Steps to run:** 1 (structure), 2 (stack), 3 (AI tools — abbreviated), 5 (git summary — abbreviated), 7 (secrets check only)

**Abbreviated Step 3 (AI tools):** Check for config files/directories only (Section 1 of reference). Check manifests for LLM SDK deps. Skip git log marker scanning and governance deep dive. Assign Adoption Level only.

**Abbreviated Step 5 (git):** Run total commit count, first/last dates, top 5 contributors, bus factor. Skip monthly breakdown and commit convention analysis.

**Report format:**

```markdown
# Quick Scan: [repo-name]
_Generated: [YYYY-MM-DD] | Depth: Quick Scan_

## At a Glance

| Aspect | Finding |
|--------|---------|
| Project Type | [monorepo / single app / library / mixed] |
| Tech Stack | [one-line summary] |
| Age / Activity | [first commit] – [last commit], ~N commits/month |
| Contributors | [count], bus factor: [🔴/🟡/🟢 assessment] |
| AI Tool Adoption | [None / Exploratory / Integrated / AI-Native] |
| Dependencies | [N total, lock file: ✅/❌] |
| Secrets Hygiene | [🔴/🟡/🟢 one-line finding] |

## Key Observations

[5-7 severity-tagged bullet points — the most important findings]

## Recommendation

[1-2 sentences: overall health assessment and whether a deeper analysis is warranted]
```

After writing:
> "Quick scan complete. Report at `<output-path>`."
> [3-4 bullet summary]
> "Run a Standard or Deep Audit for comprehensive analysis."

---

# Deep Audit Tier

When depth keywords match Deep Audit, use specialist agents for deeper analysis in AI tools, security, and infrastructure/compliance.

## Deep Audit Flow

### Phase 1: Context gathering (orchestrator)

Run Steps 1-2 (directory structure + tech stack) to establish context. Write a brief context summary to `.diligence-report/context.md`:

```bash
mkdir -p .diligence-report
```

Write `context.md` containing: repo path, project type, tech stack summary, manifest files found. This file is input for all specialist agents.

### Phase 2: Parallel analysis

Spawn 3 specialist agents **in parallel** while the orchestrator runs remaining steps itself.

**Spawn agents:**

Read `agents/ai-tools-analyst.md`. Spawn a subagent named **`ai-tools-analyst`** with:
- Path to `.diligence-report/context.md`
- Repo path
- Output path: `.diligence-report/ai-tools.md`
- Names of other agents for cross-referencing: **`security-analyst`**

Read `agents/security-analyst.md`. Spawn a subagent named **`security-analyst`** with:
- Path to `.diligence-report/context.md`
- Repo path
- Output path: `.diligence-report/security.md`
- Names of other agents: **`ai-tools-analyst`**

Read `agents/infra-compliance-analyst.md`. Spawn a subagent named **`infra-compliance-analyst`** with:
- Path to `.diligence-report/context.md`
- Repo path
- Output path: `.diligence-report/infra-compliance.md`

**While agents run**, the orchestrator executes Steps 4, 5, 6, 8, 9, 12, 13 itself (dependency health, git history, code quality, documentation, architecture, API surface, team maturity).

### Phase 3: Verify artifacts

After all agents complete, verify each artifact exists and is non-empty:
```bash
test -s .diligence-report/ai-tools.md && echo "ai-tools: ok" || echo "ai-tools: MISSING"
test -s .diligence-report/security.md && echo "security: ok" || echo "security: MISSING"
test -s .diligence-report/infra-compliance.md && echo "infra: ok" || echo "infra: MISSING"
```

If any artifact is missing, surface the failure and continue with available data.

### Phase 4: Synthesize

Read all agent artifacts. Combine with orchestrator's own findings. Write the full report using the Standard report format, plus one additional section:

**Cross-Cutting Findings** — inserted between the last detail section and Findings Summary. The synthesizer identifies patterns that span multiple analyst domains:

```markdown
## Cross-Cutting Findings

[Patterns that emerge from combining findings across domains. Examples:]
- "Heavy AI tool adoption (AI-Native) combined with weak secrets hygiene creates elevated
  risk of AI API key exposure."
- "Single contributor with 90% of commits is also the only person who understands the
  Terraform infrastructure — compounding the bus factor risk."
- "LLM dependencies are production-critical but CI/CD pipeline has no AI-specific test
  coverage or prompt regression testing."
```

The Deep Audit report header shows `Depth: Deep Audit` and all detail sections sourced from specialist agents are marked with deeper analysis than Standard would produce.

### Focus mode + Deep Audit

If focus mode is active during a Deep Audit, only spawn the relevant specialist agent(s):
- AI focus → spawn `ai-tools-analyst` only
- Security focus → spawn `security-analyst` only
- Infrastructure/licensing focus → spawn `infra-compliance-analyst` only
- Other focus areas → orchestrator handles directly (no agents needed)

---

## Agent Lifecycle

All specialist agents are spawned in **background mode** and remain reachable via `SendMessage`. An agent that finishes writing its artifact enters a waiting state — do not terminate. It may receive cross-referencing queries from other agents.

**Collaboration discipline:** Agents should message each other only when genuinely blocked by an ambiguity that would materially change their output. For example, `security-analyst` might ask `ai-tools-analyst` whether any detected AI API keys are in committed files. Do not message for minor questions that can be resolved by reading `context.md` or doing a targeted search.

---

## Calibration Notes

- **Depth scaling:** Scale exploration depth to codebase size. A small focused repo needs less investigation than a large monolith.
- **Context limits:** For very large repos, use `head` to limit command output. Note truncation in the report.
- **Failure handling:** After each agent completes, verify its artifact exists and is non-empty. If missing, note in the report and continue.
- **Transparency:** If any analysis area cannot be fully explored, name what was examined and what could not be assessed. Reports built on incomplete data should say so.
- **No modifications:** This skill is strictly read-only. Do not modify any files in the target repo. The only files written are the report and (for Deep Audit) the `.diligence-report/` artifacts.
