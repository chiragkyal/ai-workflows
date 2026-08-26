# CVE Analysis Workflow

Perform comprehensive security vulnerability analysis for Go projects. Given a CVE identifier, gather vulnerability intelligence, analyze the codebase for impact, generate a risk report, optionally apply fixes, and optionally open a GitHub pull request after a verified fix.

## Arguments

Exactly one of the following input modes is required:

- **CVE-ID** — Direct CVE identifier (format: `CVE-YYYY-NNNNN`, case-insensitive). Use when you already know the CVE.
- **--jira=PROJ-NNN** — Red Hat Jira ticket key (e.g. `--jira=OCPBUGS-12345`). The workflow fetches the ticket and extracts the CVE ID from it. Use when you are starting from a Jira issue.
- **--jql="..."** — JQL query (e.g. `--jql="project = OCPBUGS AND labels = needs-cve-analysis"`). The workflow fetches matching issues, processes **only the first result**, and logs the rest as skipped. Use when you want to pick a ticket automatically from a queue.

Optional flags:

- **--repo=\<url-or-component\>** — Repository to analyze. Accepts:
  - A full GitHub URL: `--repo=https://github.com/openshift/hypershift`
  - A short component name: `--repo=hypershift` (resolved via `component-repo-mapping` skill)
  - If omitted, the workflow checks for pre-cloned repos first, then resolves from the Jira ticket's components (if `--jira` was used), then prompts the user.
- **--algo** (default: `vta`): Call graph construction algorithm.
  - `vta` — Most precise, fewest false positives (recommended)
  - `rta` — Good balance of precision and speed
  - `cha` — Fast, less precise
  - `static` — Fastest, least precise

## Security — Credential Handling

> **This rule applies to every shell command, log line, and model response in this workflow, without exception.**

- **Never print, echo, log, or display credentials in any form.** This includes API tokens, passwords, PATs, service-account keys, and any environment variable whose name contains `TOKEN`, `KEY`, `SECRET`, `PASSWORD`, `PAT`, `CREDENTIAL`, or `AUTH`.
- If a command requires a credential, pass it directly via the environment variable reference (e.g. `$JIRA_TOKEN`). Never interpolate the value into a string that will be printed or logged.
- If a credential accidentally appears in command output, **do not repeat or quote it** in any subsequent message or log.
- When logging command invocations for debugging, **mask** credential arguments:
  ```bash
  # Good — value never appears in output
  curl -H "Authorization: Bearer $JIRA_TOKEN" ...
  echo "Calling Jira API with Bearer token (masked)"

  # Bad — token value exposed in log
  echo "Token is: $JIRA_TOKEN"
  curl -v -H "Authorization: Bearer eyJhb..."
  ```
- The same rule applies to SSH keys, `~/.netrc` contents, Git credential helpers, and any secrets mounted as files.

---

## Prerequisites

All tools below are **required**. Exit with an error if any are missing.

```bash
go install golang.org/x/vuln/cmd/govulncheck@latest
go install golang.org/x/tools/cmd/callgraph@latest
go install golang.org/x/tools/cmd/digraph@latest
```

**Optional:**
- `graphviz` for visual call graph generation (`brew install graphviz` or `sudo apt-get install graphviz`).
- `gh` (GitHub CLI) for Phase 6 pull-request creation (`https://cli.github.com/`). Authenticate with `gh auth login`. Missing `gh` does **not** fail Phase 0 — analysis and local fixes still run; Phase 6 is skipped until `gh` is available.

**Internet access** is recommended for CVE data fetching but not required if the user can provide CVE details manually.

---

## Phase 0: Setup and Tool Validation

1. **Parse Arguments**
   - Determine input mode:
     - IF `--jql="..."` provided → set `JQL_QUERY`, resolve to a single ticket in Phase 0.3
     - ELSE IF `--jira=PROJ-NNN` provided → set `JIRA_TICKET=PROJ-NNN`, CVE-ID to be resolved in Phase 0.5
     - ELSE IF a bare `CVE-YYYY-NNNNN` token is present → set `CVE_ID` directly
     - ELSE → exit with error: "Provide a CVE ID, a Jira ticket (--jira=), or a JQL query (--jql=)"
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
4. **Optional Phase 6 tool** (warn only, do not exit):

   ```bash
   which gh 2>/dev/null || echo "OPTIONAL: gh (needed only for Phase 6 GitHub PR creation)"
   ```

5. **If all required tools present** → Continue to Phase 0.3 (if JQL mode), Phase 0.5 (if Jira mode), or Phase 0.7 (if direct CVE mode).

---

## Phase 0.3: JQL Resolution _(only when `--jql` is provided)_

Run the JQL query, take the **first result only**, and continue as if `--jira=<first-ticket>` was provided.

```python
results = mcp__atlassian__jira_search_issues(
    jql=JQL_QUERY,
    start_at=0,
    max_results=10   # fetch a few to log the skipped ones
)
```

**Decision Point:**
- IF query returns 0 results → exit with: `No Jira issues matched the JQL query: <JQL_QUERY>`
- IF query returns 1 or more results:
  - Set `JIRA_TICKET = results[0].key`
  - Print a summary table of all returned issues, clearly marking which is being processed:

  ```
  JQL matched <N> issue(s). Processing the first; the rest are skipped this run.

  ✅  <PROJ-NNN>  <summary>          ← processing now
  ⏭   <PROJ-NNN>  <summary>          ← skipped
  ⏭   <PROJ-NNN>  <summary>          ← skipped
  ...
  ```

- Continue to Phase 0.5 with `JIRA_TICKET` set.

---

## Phase 0.5: Jira CVE Extraction _(only when `--jira` or `--jql` is provided)_

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

### Storage path

Cloned repos are stored under the **persistent workflow mount** so they survive Ambient shell resets:

```bash
REPOS_BASE="/workspace/workflows/ai-workflows/.work/repos"
mkdir -p "${REPOS_BASE}"
```

`/workspace/workflows/` is the persistent git mount. `.work/` is gitignored. Repos cloned here are **not** wiped when the container shell recycles (unlike `/workspace/repos/` which is ephemeral scratch).

### Step 1: Check for Pre-Cloned Repository

Check both the persistent path and the ephemeral path (Ambient pre-clones via the session `repos` field into `/workspace/repos/`):

```bash
# Check persistent path first, then ephemeral
for CHECK_DIR in "${REPOS_BASE}" /workspace/repos; do
  ls "${CHECK_DIR}/" 2>/dev/null
done
```

- IF either path contains exactly one directory → use it as `REPO_DIR`, skip to Step 3.
- IF either path contains multiple directories → list them, ask user which to analyse.
- IF both are empty → continue to Step 2.

### Step 2: Resolve Repository URL

Determine `REPO_URL` using the first applicable source:

1. **`--repo` flag provided**:
   - Full URL (`https://...`) → use directly.
   - Short name or image name → run `image-repo-mapping` skill.
2. **`--jira` was used and `IMAGE_NAME` was extracted** → run `image-repo-mapping` skill with `IMAGE_NAME`.
3. **Neither** → prompt user: "Please provide the repository URL or image name (e.g. https://github.com/openshift/cert-manager-operator or --repo=cert-manager-operator-rhel9)."

**Decision Point:**
- IF `REPO_URL` still unresolved after prompting → Exit with error.

### Step 3: Resolve the Target Branch and Repository Pattern

Check whether the image maps to a **Pattern A** (direct repo) or **Pattern B** (release repo + submodules) component — this is indicated in the `image-repo-mapping` skill output.

**If Pattern B:** the `REPO_URL` returned by `image-repo-mapping` is the release repo URL. Continue to Step 3a (release repo clone), then Step 3c (submodule resolution), then Step 3b (component clone).

**If Pattern A:** skip Steps 3a and 3c. Proceed directly to Step 3b with `GIT_BRANCH` from Step 3 below.

### Step 3a (Pattern B only): Clone the Release Repo and Read `.gitmodules`

Use `BRANCH` extracted by `jira-cve-extraction` (e.g. `openshift-4.17`, `ztwim-1.0`).

**Branch name mapping** — Jira ticket summaries use a different naming convention from the actual git branches:

| Jira `BRANCH` value     | Component group                                                   | Git branch                                                                |
| ----------------------- | ----------------------------------------------------------------- | ------------------------------------------------------------------------- |
| `openshift-X.Y`         | Operator SDK, Ansible, must-gather, Secrets Store CSI (Pattern A) | `release-X.Y`                                                             |
| `openshift-X.Y.z`       | (same Pattern A components)                                       | `release-X.Y.z`                                                           |
| `cert-manager-X-Y`      | cert-manager (Pattern B release repo)                             | `release-X.Y`                                                             |
| `external-secrets-X-Y`  | ESO (Pattern B release repo)                                      | `release-X.Y`                                                             |
| `ztwim-1.0`             | ZTWIM (Pattern B release repo)                                    | `release-1.0.0` _(one-time exception; future releases use `release-X.Y`)_ |
| `ztwim-X.Y` (any other) | ZTWIM (Pattern B release repo)                                    | `release-X.Y`                                                             |
| Any other value         | —                                                                 | Use verbatim                                                        |

**Verify the branch exists before cloning:**

```bash
git ls-remote --heads "${REPO_URL}" "${GIT_BRANCH}" | grep -q "${GIT_BRANCH}"
```

- IF branch exists → clone with `-b "${GIT_BRANCH}"` in Step 3b.
- IF branch does not exist → try the Jira value verbatim as a fallback, then warn user and ask to confirm or provide the correct branch name.
- IF `BRANCH` was not extracted (direct CVE mode, no Jira ticket) → clone default branch; note this in the analysis.

### Step 3c (Pattern B only): Read `.gitmodules` and Resolve Component Repo

```bash
# Clone release repo (shallow, no submodule content needed)
echo "Cloning release repo ${RELEASE_REPO_URL} @ ${GIT_BRANCH} ..."
timeout 120 git clone --depth=1 -b "${GIT_BRANCH}" "${RELEASE_REPO_URL}" /tmp/release-repo
if [ $? -eq 124 ]; then
  echo "ERROR: release repo clone timed out after 120s"
  exit 1
fi
echo "✓ Release repo cloned"

# Print .gitmodules so the model can parse it
cat /tmp/release-repo/.gitmodules
```

From `.gitmodules`, find the entry matching the target image (use the submodule name from the `image-repo-mapping` skill output). Extract:
- `url` → the component repo to clone (`COMPONENT_URL`)
- `branch` → clone at this branch (`COMPONENT_BRANCH`)
- `tag` → if present instead of `branch`, clone at this tag (`COMPONENT_TAG`)

Set `REPO_URL = COMPONENT_URL` and `GIT_BRANCH = COMPONENT_BRANCH` (or `COMPONENT_TAG`) for Step 3b.

### Step 3b: Clone the Repository

```bash
REPO_NAME=$(basename "${REPO_URL}" .git)
REPO_DIR="${REPOS_BASE}/${REPO_NAME}"

if [ ! -d "${REPO_DIR}/.git" ]; then
  echo "Cloning ${REPO_URL} (branch: ${GIT_BRANCH:-default}) into ${REPO_DIR} ..."
  mkdir -p "${REPOS_BASE}"
  if [ -n "${GIT_BRANCH}" ]; then
    timeout -k 10 300 git clone --depth=50 -b "${GIT_BRANCH}" "${REPO_URL}" "${REPO_DIR}"
  else
    timeout -k 10 300 git clone --depth=50 "${REPO_URL}" "${REPO_DIR}"
  fi
  CLONE_EXIT=$?
  if [ $CLONE_EXIT -eq 124 ] || [ $CLONE_EXIT -eq 137 ]; then
    echo "ERROR: git clone timed out after 300s for ${REPO_URL}"
    echo "The repository may be too large or the network too slow."
    exit 1
  elif [ $CLONE_EXIT -ne 0 ]; then
    echo "ERROR: git clone failed (exit ${CLONE_EXIT}) for ${REPO_URL}"
    exit 1
  fi
else
  CURRENT_BRANCH=$(git -C "${REPO_DIR}" rev-parse --abbrev-ref HEAD)
  if [ -n "${GIT_BRANCH}" ] && [ "${CURRENT_BRANCH}" != "${GIT_BRANCH}" ]; then
    echo "Switching from ${CURRENT_BRANCH} to ${GIT_BRANCH} ..."
    timeout -k 10 120 git -C "${REPO_DIR}" fetch origin "${GIT_BRANCH}"
    git -C "${REPO_DIR}" checkout "${GIT_BRANCH}"
  fi
  timeout -k 10 120 git -C "${REPO_DIR}" pull --ff-only
fi

# Explicit verification — always print this so it's visible in the session
echo "✓ Repository ready: ${REPO_DIR}"
echo "  Branch : $(git -C "${REPO_DIR}" rev-parse --abbrev-ref HEAD)"
echo "  Commit : $(git -C "${REPO_DIR}" rev-parse --short HEAD)"
echo "  go.mod : $([ -f "${REPO_DIR}/go.mod" ] && echo 'present' || echo 'MISSING')"
```

- IF clone times out → exit with instructions to pre-clone or retry.
- IF clone fails → exit with error details.
- IF clone succeeds → `REPO_DIR` and `GIT_BRANCH` are set as working context for all subsequent phases.
- The verification block at the end **must always print** — this confirms to the user that the repo is ready and is visible in the session context.

### Step 4: Verify Go Project

```bash
[ -f "${REPO_DIR}/go.mod" ] || echo "WARNING: no go.mod found in ${REPO_DIR}"
```

- IF `go.mod` missing → warn user; call graph and govulncheck steps will be skipped, dependency-based methods only.
- IF `go.mod` present → Continue to Phase 1.

### Repo Guard — Re-clone if Missing

Although repos are now cloned to the persistent workflow mount (`.work/repos/`), edge cases like disk pressure or manual cleanup can still cause loss. Every phase that needs `REPO_DIR` must run this guard first:

```bash
if [ ! -f "${REPO_DIR}/go.mod" ]; then
  echo "⚠ Repo missing at ${REPO_DIR} — re-cloning..."
  mkdir -p "${REPOS_BASE}"
  if [ -n "${GIT_BRANCH}" ]; then
    timeout -k 10 300 git clone --depth=50 -b "${GIT_BRANCH}" "${REPO_URL}" "${REPO_DIR}"
  else
    timeout -k 10 300 git clone --depth=50 "${REPO_URL}" "${REPO_DIR}"
  fi
  if [ $? -ne 0 ]; then
    echo "✗ Re-clone failed. Cannot continue without the repository."
    exit 1
  fi
  echo "✓ Re-cloned: ${REPO_DIR} @ $(git -C "${REPO_DIR}" rev-parse --abbrev-ref HEAD)"
fi
```

Run this guard at the start of **Phase 2, Phase 4, Phase 5, and Phase 6**.

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

**Before starting:** Run the [Repo Guard](#repo-guard--re-clone-if-missing) to verify `REPO_DIR` still exists. Re-clone if needed.

- **Skill**: [codebase-impact-analysis](skills/codebase-impact-analysis/SKILL.md)
  - Sub-skill: [call-graph-analysis](skills/call-graph-analysis/SKILL.md)
- **Working directory**: `REPO_DIR` set in Phase 0.7 (e.g. `.work/repos/hypershift`)
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
- **Output**: comment and label posted to `SOURCE_TICKET` (the same ticket the CVE details were read from); skipped silently if no `--jira` was provided; if posting fails, comment body is displayed in session for manual copy-paste

---

## Phase 5: Interactive Fix Application

**Before starting:** Run the [Repo Guard](#repo-guard--re-clone-if-missing) to verify `REPO_DIR` still exists. Re-clone if needed.

Requires **explicit user approval** before proceeding. Do not change live cluster or production-environment configuration. Repo-tracked config files are allowed only as the approved remediation.

**Before applying anything**, snapshot the worktree so Phase 6 can stage only Phase 5 files (including new untracked paths). `WORK_CVE` is in the **workflow workspace**, not inside `REPO_DIR`:

```bash
WORK_CVE=".work/compliance/analyze-cve/${CVE_ID}"
mkdir -p "${WORK_CVE}"
git -C "${REPO_DIR}" status --porcelain > "${WORK_CVE}/phase5-before.status"
```

1. **Apply Fixes**
   - Dependency bump: update `go.mod`/`go.sum` with `go get -u <package>@<fixed-version>` + `go mod tidy`
   - **Vendor sync (dependency bumps only):** IF `go.mod`/`go.sum` changed **and** `vendor/` exists → run `go mod vendor` (or `make vendor` if that target exists) now, in Phase 5, before writing `PHASE5_FILES`. Phase 6 no longer runs vendoring — if it did, the generated `vendor/` paths would be missing from the allowlist and silently dropped from the commit.
   - Source changes if required (as identified in Phase 4)
   - Repo-tracked config (YAML, Dockerfiles, scripts) if that is the approved remediation

2. **Verify Changes** (after vendor sync, so the vendored tree is what gets verified)
   - Check for Makefile targets first, fall back to standard Go commands:
     - Verify: `make verify` or `go mod verify`
     - Build: `make build` or `go build ./...`
     - Test: `make test` or `go test ./...`
   - Re-check: `govulncheck ./...`

3. **Document Changes**
   - Summary of changes, files modified, git diff, suggested commit message (Phase 6 uses this if the user approves a PR)
   - Write `PHASE5_FILES` (one **repo-relative** path per line, no porcelain status prefix) to `${WORK_CVE}/phase5-files.txt`. Include every path Phase 5 added, modified, or deleted — including untracked files, and every `vendor/` path touched by the sync above. Union of:
     - porcelain-status paths that are new or whose status code changed vs `phase5-before.status`
     - paths this phase actually edited (so a pre-dirty file Phase 5 touched is not dropped)
   - Exclude `.work/`, analysis reports, and credentials. Do not list pre-existing dirty files that Phase 5 did not touch.

**Decision Point:**
- IF verification failed → stop. Do not offer a PR. Leave the tree for the user to inspect.
- IF verification succeeded → Continue to Phase 6.

---

## Phase 6: GitHub PR Creation

**Before starting:** Run the [Repo Guard](#repo-guard--re-clone-if-missing) to verify `REPO_DIR` still exists. Re-clone if needed.

- **Skill**: [create-fix-pr](skills/create-fix-pr/SKILL.md)
- **Input**: `REPO_DIR`, `GIT_BRANCH`, `REPO_URL`, `CVE_ID`, `SOURCE_TICKET` (if `--jira` was provided), `PHASE5_FILES` allowlist, Phase 5 change summary, and module bump (`old` → `new`) **only if** the fix is a dependency bump
- **Output**: GitHub PR URL (created or updated); optional follow-up Jira comment with that URL

Requires **explicit user approval** before any commit, push, or `gh pr create`. This is a separate approval from Phase 5 (applying the fix locally does not imply opening a PR).

1. Ask: "The fix is applied and verified locally. Create a GitHub PR against `<GIT_BRANCH>`?"
2. IF no → Exit. Leave local changes uncommitted (or committed only if the user asked). Print the suggested commit message from Phase 5.
3. IF yes → Run `create-fix-pr` (`git` and `gh` are **hard requirements of that skill** — if either is missing or `gh` is unauthenticated, fail Phase 6; do not create the PR another way):
   - Check open PRs on the same `org/repo` + base branch whose **title** contains this `CVE_ID` or `SOURCE_TICKET` (title only — ignore files, body, and module versions)
   - If a title match exists, present **stack / wait / independent** and wait for the user; do not guess
   - Branch from the mapped release branch, commit **only `PHASE5_FILES`** (for a version bump that is often `go.mod` / `go.sum` / `vendor/`, already vendor-synced in Phase 5; other remediations may be source or config only) with `UPSTREAM:` commit style when it applies, and `--signoff`
   - Validate the staged (and, for `stack`, the branch) path set exactly matches `PHASE5_FILES`; reject/unstage anything extra before continuing
   - Push and `gh pr create` (or update the stacked PR)
   - PR title/body include `CVE_ID`, a summary of the actual Phase 5 change (module/version only when the fix is a dependency bump), short CVE description, and — in Jira mode — `Fixes: [TICKET](https://redhat.atlassian.net/browse/TICKET)`
   - Direct CVE mode (no `--jira`): create the PR **without** Jira links
4. After the PR exists, post the PR URL as a **new** comment on `SOURCE_TICKET` (do not replace the Phase 4 analysis comment). Skip Jira posting in direct CVE mode.
5. Embargo abort and "never change code without approval" still apply. Never force-push `release-*` / `main`.

---

## Output

- **Format**: Markdown report at `.work/compliance/analyze-cve/{CVE-ID}/report.md`
- **Content**: Vulnerability details, risk assessment, evidence, remediation recommendations, applied fixes (if approved), GitHub PR URL (if Phase 6 ran)

## Notes

- Focuses on Go-specific vulnerabilities.
- Falls back to user-provided information if internet access fails.
- Does NOT make changes, commits, or pull requests without explicit user approval.
- Reports are saved locally and not committed to git.
