# CVE Analysis Workflow

Perform comprehensive security vulnerability analysis for Go projects. Given a CVE identifier, gather vulnerability intelligence, analyze the codebase for impact, generate a risk report, and optionally apply fixes.

## Arguments

Exactly one of the following input modes is required:

- **CVE-ID** — Direct CVE identifier (format: `CVE-YYYY-NNNNN`, case-insensitive). Use when you already know the CVE.
- **--jira=PROJ-NNN** — Red Hat Jira ticket key (e.g. `--jira=OCPBUGS-12345`). The workflow fetches the ticket and extracts the CVE ID from it. Use when you are starting from a Jira issue.

Optional flags:

- **--repo=\<url-or-component\>** — Repository to analyze. Accepts:
  - A full GitHub URL: `--repo=https://github.com/openshift/hypershift`
  - A short component name: `--repo=hypershift` (resolved via `component-repo-mapping` skill)
  - If omitted, the workflow checks `/workspace/repos/` first, then resolves from the Jira ticket's components (if `--jira` was used), then prompts the user.
- **--algo** (default: `vta`): Call graph construction algorithm.
  - `vta` — Most precise, fewest false positives (recommended)
  - `rta` — Good balance of precision and speed
  - `cha` — Fast, less precise
  - `static` — Fastest, least precise

## Prerequisites

All tools below are **required**. Exit with an error if any are missing.

```bash
go install golang.org/x/vuln/cmd/govulncheck@latest
go install golang.org/x/tools/cmd/callgraph@latest
go install golang.org/x/tools/cmd/digraph@latest
```

**Optional:** `graphviz` for visual call graph generation (`brew install graphviz` or `sudo apt-get install graphviz`).

**Internet access** is recommended for CVE data fetching but not required if the user can provide CVE details manually.

---

## Phase 0: Setup and Tool Validation

1. **Parse Arguments**
   - Determine input mode:
     - IF `--jira=PROJ-NNN` provided → set `JIRA_TICKET=PROJ-NNN`, CVE-ID to be resolved in Phase 0.5
     - ELSE IF a bare `CVE-YYYY-NNNNN` token is present → set `CVE_ID` directly
     - ELSE → exit with error: "Provide either a CVE ID (e.g. CVE-2024-45338) or a Jira ticket (e.g. --jira=OCPBUGS-12345)"
   - Extract `--repo` value if provided (optional); store as `REPO_INPUT`.
   - Extract `--algo` value if provided (optional, default: `vta`).
   - Valid `--algo` values: `vta`, `rta`, `cha`, `static`.

2. **Check Required Tools**

   ```bash
   go version 2>/dev/null || echo "MISSING: go"
   which govulncheck 2>/dev/null || echo "MISSING: govulncheck"
   which callgraph 2>/dev/null || echo "MISSING: callgraph"
   which digraph 2>/dev/null || echo "MISSING: digraph"
   which git 2>/dev/null || echo "MISSING: git"
   ```

3. **If ANY tool is missing** → Display installation instructions and **exit with error**.
4. **If all tools present** → Continue to Phase 0.5 (if Jira mode) or Phase 0.7 (if direct CVE mode).

---

## Phase 0.5: Jira CVE Extraction _(only when `--jira` is provided)_

- **Skill**: [jira-cve-extraction](skills/jira-cve-extraction/SKILL.md)
- **References**: [`reference/jira-mcp-tools.md`](reference/jira-mcp-tools.md), [`reference/jira-cli-fallback.md`](reference/jira-cli-fallback.md)
- **Input**: Jira ticket key from `--jira` argument
- **Output**: `CVE_ID`, `IMAGE_NAME`, `BRANCH`, and full `jira_context` enrichment block

**Extraction priority (per skill):**
1. Parse ticket **summary** — format `CVE-YYYY-NNNNN <image>: <desc> [branch]` — provides CVE ID, image name, and branch in one step
2. `pscomponent:` label → image name fallback
3. `Downstream Component Name` custom field → image name fallback
4. `Custom field (CVE ID)` → CVE ID fallback

**Decision Point:**
- IF ticket not found or access denied → Exit with error
- IF `embargo_status = True` → **Exit immediately. Do not proceed. Do not output any ticket data.**
- IF no CVE ID found → Prompt user to supply manually; if declined → Exit
- IF no image name found → Leave blank; Phase 0.7 will prompt
- IF resolved → Set `CVE_ID` + `IMAGE_NAME`, carry `jira_context` (includes CVSS, CWE, priority, versions) forward → Continue to Phase 0.7

---

## Phase 0.7: Repository Resolution and Cloning

- **Skill**: [image-repo-mapping](skills/image-repo-mapping/SKILL.md)
- **Working directory for all subsequent phases**: `/workspace/repos/<repo-name>`

### Step 1: Check for Pre-Cloned Repository

```bash
ls /workspace/repos/ 2>/dev/null
```

- IF `/workspace/repos/` contains exactly one directory → use it as `REPO_DIR`, skip to Step 3.
- IF `/workspace/repos/` contains multiple directories → list them, ask user which to analyse.
- IF `/workspace/repos/` is empty → continue to Step 2.

### Step 2: Resolve Repository URL

Determine `REPO_URL` using the first applicable source:

1. **`--repo` flag provided**:
   - Full URL (`https://...`) → use directly.
   - Short name or image name → run `image-repo-mapping` skill.
2. **`--jira` was used and `IMAGE_NAME` was extracted** → run `image-repo-mapping` skill with `IMAGE_NAME`.
3. **Neither** → prompt user: "Please provide the repository URL or image name (e.g. https://github.com/openshift/cert-manager-operator or --repo=cert-manager-operator-rhel9)."

**Decision Point:**
- IF `REPO_URL` still unresolved after prompting → Exit with error.

### Step 3: Clone the Repository

```bash
REPO_NAME=$(basename "${REPO_URL}" .git)
REPO_DIR="/workspace/repos/${REPO_NAME}"

if [ ! -d "${REPO_DIR}/.git" ]; then
  echo "Cloning ${REPO_URL} into ${REPO_DIR} ..."
  git clone --depth=50 "${REPO_URL}" "${REPO_DIR}"
else
  echo "Repository already present at ${REPO_DIR}, pulling latest..."
  git -C "${REPO_DIR}" pull --ff-only
fi
```

- IF clone fails (auth error, not found) → Display error with fix instructions and Exit.
- IF clone succeeds → set `REPO_DIR` as working directory for all subsequent phases.

### Step 4: Verify Go Project

```bash
[ -f "${REPO_DIR}/go.mod" ] || echo "WARNING: no go.mod found in ${REPO_DIR}"
```

- IF `go.mod` missing → warn user; call graph and govulncheck steps will be skipped, dependency-based methods only.
- IF `go.mod` present → Continue to Phase 1.

---

## Phase 1: CVE Intelligence Gathering

- **Skill**: [cve-intelligence-gathering](skills/cve-intelligence-gathering/SKILL.md)
- **Input**: CVE-ID from user input **+ `jira_context` from Phase 0.5 (if `--jira` was provided)**
- **Output**: Merged CVE profile combining Jira internal data with public sources (NVD, GHSA, Go vulndb)

Pass the full `jira_context` object from Phase 0.5 into the skill. The skill uses Jira fields (CVSS, CWE, affected versions, internal notes, workarounds, release note text) as a pre-populated starting point, then uses web searches to verify, fill gaps, and add public context. Neither source replaces the other — both are combined.

**Decision Point:**
- IF invalid CVE format → Exit with error
- IF CVE not found AND user declines to provide info → Exit with error
- IF CVE is not Go-related → Generate "Not Applicable" report → Exit
- IF CVE details found → Continue to Phase 2

---

## Phase 2: Codebase Impact Analysis

- **Skill**: [codebase-impact-analysis](skills/codebase-impact-analysis/SKILL.md)
  - Sub-skill: [call-graph-analysis](skills/call-graph-analysis/SKILL.md)
- **Working directory**: `REPO_DIR` set in Phase 0.7 (e.g. `/workspace/repos/hypershift`)
- **Input**: CVE profile from Phase 1, `--algo` preference
- **Output**: Risk level (HIGH/MEDIUM/LOW/NEEDS_REVIEW), evidence package, confidence assessment

**Decision Point:**
- IF HIGH RISK or MEDIUM RISK → Generate report (Phase 3) → Proceed to Phase 4
- IF LOW RISK → Generate report (Phase 3) → Recommend manual review → Exit
- IF NEEDS REVIEW → Generate report (Phase 3) → Ask user if they want remediation guidance
  - IF yes → Proceed to Phase 4
  - IF no → Exit

---

## Phase 3: Report Generation

Generate analysis report at `.work/compliance/analyze-cve/{CVE-ID}/report.md`.

**Report structure:**
- Executive Summary: risk level, confidence, key takeaway
- CVE Context: vulnerability description, sources (tag verified vs user-provided)
- Jira Context _(if `--jira` was provided)_: ticket URL, priority, status, assignee, target versions, components, internal notes, linked issues
- Analysis Methods: what was used, why, and what was found
- Findings: specific evidence (file paths, versions, code snippets, call chains)
- Risk Assessment: severity + actual exposure + exploitability in this context; escalate if `jira_context.analysis_hints.urgency_override` is set
- Next Steps: remediation guidance or monitoring recommendations; note any existing workarounds from the Jira ticket
- Sources and Limitations: tools used, gaps, analysis date

**Additional artifacts** (as generated):
- `callgraph.svg` (if call graph analysis was performed)
- `govulncheck-output.txt` (if scanner was run)
- `evidence.json` (structured evidence data)

---

## Phase 4: Remediation Guidance

- **Skill**: [remediation-planning](skills/remediation-planning/SKILL.md)
- **Input**: CVE profile from Phase 1, risk level and evidence from Phase 2
- **Output**: Remediation plan (strategy, commands, verification steps, risk assessment)

**Decision Point:**
- Present remediation plan to user
- Ask: "Would you like me to apply these fixes automatically?"
- IF yes → Continue to Phase 5
- IF no → Exit with report and manual instructions

After presenting the report (regardless of whether the user proceeds to Phase 5), invoke the report-to-jira skill:

- **Skill**: [report-to-jira](skills/report-to-jira/SKILL.md)
- **Input**: completed report, CVE ID, risk level, repo URL, source Jira ticket key (if `--jira` was provided)
- **Output**: comment posted to OAPE-751; if posting fails, comment body is displayed in session for manual copy-paste

---

## Phase 5: Interactive Fix Application

Requires **explicit user approval** before proceeding.

1. **Apply Fixes**
   - Update `go.mod`/`go.sum`: `go get -u <package>@<fixed-version>` + `go mod tidy`
   - Modify source code if required (as identified in Phase 4)

2. **Verify Changes**
   - Check for Makefile targets first, fall back to standard Go commands:
     - Verify: `make verify` or `go mod verify`
     - Build: `make build` or `go build ./...`
     - Test: `make test` or `go test ./...`
   - Re-check: `govulncheck ./...`

3. **Document Changes**
   - Summary of changes, files modified, git diff, suggested commit message

---

## Output

- **Format**: Markdown report at `.work/compliance/analyze-cve/{CVE-ID}/report.md`
- **Content**: Vulnerability details, risk assessment, evidence, remediation recommendations, applied fixes (if approved)

## Notes

- Focuses on Go-specific vulnerabilities.
- Falls back to user-provided information if internet access fails.
- Does NOT make changes without explicit user approval.
- Reports are saved locally and not committed to git.
