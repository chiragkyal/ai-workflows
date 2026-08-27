---
name: create-fix-pr
description: After a CVE fix is applied and verified locally, create (or update) a GitHub pull request and optionally post the PR URL back to the source Jira ticket
---

# Create Fix PR

Opens a GitHub pull request for the CVE fix already applied in `REPO_DIR` (Phase 5). Never commits, pushes, or opens a PR without **explicit approval** — interactive, or `AUTO_APPROVE=yes` recorded once upfront (see [Autonomous Mode](../../CLAUDE.md#autonomous-mode---auto-approveyesno)). Direct CVE mode (no `--jira`) still creates the PR; it just omits Jira `Fixes:` links and the Jira follow-up comment.

Called from **Phase 6** of the parent workflow after Phase 5 verify succeeds.

---

## When to Use This Skill

Use this skill when:

- Phase 5 applied a fix in `REPO_DIR` and verification passed
- The user approved creating a GitHub PR
- `embargo_status` is not `True`

Do **not** use this skill to apply the fix itself (that is Phase 5) or to post the analysis report (that is `report-to-jira` in Phase 4).

---

## Required Inputs

From the parent workflow:

| Input                                 | Source                                                 | Required                 |
| ------------------------------------- | ------------------------------------------------------ | ------------------------ |
| `REPO_DIR`                            | Phase 0.7                                              | Yes                      |
| `GIT_BRANCH`                          | Phase 0.7 (mapped release branch, e.g. `release-4.21`) | Yes                      |
| `REPO_URL`                            | Phase 0.7                                              | Yes                      |
| `CVE_ID`                              | Phase 0.5 / direct CVE mode                            | Yes                      |
| `SOURCE_TICKET`                       | `--jira=` / `--jql=`                                   | No                       |
| `AUTO_APPROVE`                        | Phase 0 (`--auto-approve`, default `no`)               | Yes                      |
| `PHASE5_FILES`                        | Phase 5 allowlist (incl. untracked)                    | Yes                      |
| Module path, old version, new version | Phase 4 / 5                                            | Yes if a dependency bump |
| Short CVE description                 | Phase 1                                                | Recommended              |
| Phase 5 change summary                | Phase 5 Document Changes                               | Yes                      |

---

## Prerequisites

Hard requirements **for this skill**. Phase 0 does **not** treat `gh` as required (warn-only). This skill must not assume either tool is present — re-check both, then **fail** if either is missing. Do not create a PR another way.

```bash
which git 2>/dev/null || echo "MISSING: git"
which gh 2>/dev/null || echo "MISSING: gh"
gh auth status 2>/dev/null || echo "MISSING: gh auth"
```

- IF `git` is missing → print install instructions and return `status: failed` (`git_missing`). Stop. Do not commit or push.
- IF `gh` is missing → print install instructions (`https://cli.github.com/`) and return `status: failed` (`gh_missing`). Stop. Do not commit or push.
- IF `gh auth status` fails → print `gh auth login` instructions and return `status: failed` (`gh_unauthenticated`). Stop. Do not commit or push.
- Do not print token or login output that contains credentials.

> **Credential rule:** Never print, echo, or log tokens, PATs, or `gh` auth output that contains credentials. Pass credentials only via environment variable references.

---

## Step 0: Safety Gates

1. IF `embargo_status = True` → **abort immediately**. Do not commit, push, create a PR, or mention ticket contents.
2. IF Phase 5 did not complete successfully → return `status: skipped` (`phase5_incomplete`).
3. Confirm there are local changes to commit:

   ```bash
   git -C "${REPO_DIR}" status --porcelain
   git -C "${REPO_DIR}" diff --stat
   ```

   - IF working tree is clean **and** HEAD is already on a fix branch with the Phase 5 remediation committed (dependency bump, source, or config) → **validate that branch's changes against `PHASE5_FILES`** (see the check in Step 3b) before skipping to Step 4. A clean worktree only means nothing is *uncommitted*; it does not prove the existing commit(s) contain only Phase 5 paths. IF the branch diff vs `${BASE_BRANCH}` contains paths outside `PHASE5_FILES` → **always return `status: failed` (`phase5_files_mismatch`) immediately. Never gated by `AUTO_APPROVE` and never prompt** — regardless of whether `AUTO_APPROVE` is `yes` or `no`, do not ask the user and do not continue; silently including or dropping the extra committed paths is not an acceptable outcome either way.
   - IF working tree is clean and HEAD is still `${GIT_BRANCH}` with no Phase 5 commit → return `status: skipped` (`no_local_changes`).
4. IF `AUTO_APPROVE=no` (and the parent hasn't already recorded a yes) → Ask:

   ```
   Phase 5 applied the fix locally. Create a GitHub PR against <GIT_BRANCH>?
   ```

   - IF no → return `status: skipped` (`user_declined`). Leave the working tree as-is.
   - IF yes → continue. Still do not commit until Step 3.

   IF `AUTO_APPROVE=yes` → treat as yes, skip the prompt, continue. Still do not commit until Step 3.

---

## Step 1: Resolve GitHub Repo and Base Branch

```bash
git -C "${REPO_DIR}" remote get-url origin
```

Parse `ORG/REPO` from the origin URL (`github.com/openshift/hypershift.git` → `openshift/hypershift`). If origin is SSH (`git@github.com:org/repo.git`), parse the same way.

`BASE_BRANCH` = `GIT_BRANCH` from Phase 0.7 (the mapped release branch the clone is on). Do **not** open the PR against `main` unless that is actually `GIT_BRANCH`.

---

## Step 2: Detect Conflicting Open PRs

**Title match only.** Do not inspect PR files, body, module versions, or `go.mod` diffs.

List open PRs targeting the same base branch:

```bash
gh pr list --repo "${ORG}/${REPO}" --state open --base "${BASE_BRANCH}" \
  --json number,title,url,headRefName,author
```

A PR is a **match** if its **title** contains (case-insensitive) either:

- this `CVE_ID`, or
- `SOURCE_TICKET` (when set)

IF none → continue to Step 3 with strategy `new`.

IF any match:

- **`AUTO_APPROVE=no`** → present the list and wait for the user:

  ```text
  Open PR(s) on <ORG/REPO> base <BASE_BRANCH> already have this CVE/Jira in the title:

    #<N>  <title>  <url>  head=<branch>

  How should I proceed?
    1. stack     — check out that PR branch, commit this fix on top, push (PR updates)
    2. wait      — stop; do not open a competing PR
    3. independent — new branch from <BASE_BRANCH> (may need rebase later)
  ```

  | Choice        | Action                                                                                                                                                                                                                           |
  | ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | `stack`       | `git fetch origin <headRefName> && git checkout <headRefName> && git pull`. Re-apply leftover Phase 5 changes if they are not already on that branch (same method as Phase 5 — bump, patch, or config edit). Strategy = `stack`. |
  | `wait`        | Return `status: skipped` (`user_wait_for_pr`) including the existing PR URL.                                                                                                                                                     |
  | `independent` | Warn that a second PR on the same base may need rebase later. If `go.mod` is in this diff, mention possible module-file conflicts. Strategy = `new`.                                                                             |
  | No answer     | **Do not guess.** Ask again.                                                                                                                                                                                                     |

- **`AUTO_APPROVE=yes`** → **always `wait`**, automatically, with no exceptions: return `status: skipped` (`user_wait_for_pr_auto`) including the existing PR URL. Never auto-choose `stack` (that pushes commits onto a branch nobody reviewed for this run) or `independent` (that opens a second, possibly duplicate/conflicting PR) unattended. Log clearly that this run stopped because of the existing PR, so a human can revisit with `--auto-approve=no` if a different resolution is actually wanted.

---

## Step 3: Branch, Commit, Push

Work only inside `REPO_DIR`.

**Vendor:** Phase 5 already ran `go mod vendor`/`make vendor` (if applicable) and included any resulting paths in `PHASE5_FILES` before verification. Do **not** run vendor sync here — doing so after `PHASE5_FILES` was written would generate paths that never make it into the allowlist and get silently dropped from the commit. IF a dependency bump is present in `PHASE5_FILES` but `vendor/` looks out of sync (e.g. `go mod vendor` reports diffs) → stop and tell the user to re-run Phase 5, rather than fixing it here.

### 3a. Branch

**New PR (`strategy = new`):**

```bash
git -C "${REPO_DIR}" fetch origin "${BASE_BRANCH}"
git -C "${REPO_DIR}" checkout "${BASE_BRANCH}"
git -C "${REPO_DIR}" pull --ff-only origin "${BASE_BRANCH}"

SLUG="$(echo "${CVE_ID}" | tr '[:upper:]' '[:lower:]')"
if [ -n "${SOURCE_TICKET}" ]; then
  BRANCH_NAME="fix/${SLUG}-$(echo "${SOURCE_TICKET}" | tr '[:upper:]' '[:lower:]')"
else
  BRANCH_NAME="fix/${SLUG}"
fi

git -C "${REPO_DIR}" checkout -b "${BRANCH_NAME}"
```

Re-apply the Phase 5 changes on this branch if checking out `BASE_BRANCH` discarded uncommitted work. Prefer not discarding: stash before checkout if the working tree is dirty, then stash pop onto `BRANCH_NAME`.

**Stack (`strategy = stack`):** already on the existing PR branch. Set `BRANCH_NAME` to that head ref. Do not create a new branch.

### 3b. Commit

Stage **only `PHASE5_FILES`**, not the whole worktree. That allowlist is written in Phase 5 (`phase5-files.txt`) and includes new untracked files. `go.mod`, `go.sum`, `go.work`, and `vendor/` are common for **dependency version bumps**, but other CVE remediations may only touch source, config, Dockerfiles, or scripts.

`WORK_CVE` is in the **workflow workspace**, not inside `REPO_DIR`. Paths in the allowlist are relative to `REPO_DIR`.

```bash
WORK_CVE=".work/compliance/analyze-cve/${CVE_ID}"
PHASE5_FILES="${WORK_CVE}/phase5-files.txt"
```

IF `phase5-files.txt` is missing or empty → rebuild it from Phase 5 Document Changes (and `phase5-before.status` if present). IF the allowlist still cannot be determined → **always return `status: failed` (`phase5_files_missing`) immediately. Never gated by `AUTO_APPROVE` and never prompt** — regardless of `AUTO_APPROVE`, do not ask the user and do not continue. Do **not** fall back to whole-worktree `git diff` / `git add -A` / `git add .` in any case.

```bash
# inspect current tree for sanity, but do not use it as the stage set
git -C "${REPO_DIR}" status --porcelain

while IFS= read -r f; do
  [ -n "${f}" ] || continue
  git -C "${REPO_DIR}" add -- "${f}"
done < "${PHASE5_FILES}"
```

`git add -- <path>` stages modifications, deletions, and untracked files. Examples of what belongs on the allowlist:

- Dependency bump: `go.mod`, `go.sum`, and `go.work` / `vendor/` only if Phase 5 changed them
- Code/config fix: the source or config files Phase 5 added or edited
- Mixed: the union of those paths

Do **not** add `.work/`, analysis reports, or credentials. Do **not** add pre-existing dirty files that are absent from `PHASE5_FILES`. Do **not** `git add` `go.mod` / `vendor/` unless they are on the allowlist.

**Validate the staged set exactly matches the allowlist.** The loop above only *adds* allowlisted paths — it does not prove the index is free of anything else (e.g. leftover pre-existing staged files). Diff the two sets and reject extras instead of committing them silently:

```bash
git -C "${REPO_DIR}" diff --cached --name-only > "${WORK_CVE}/staged-files.txt"

EXTRA="$(comm -23 <(sort "${WORK_CVE}/staged-files.txt") <(sort "${PHASE5_FILES}"))"
if [ -n "${EXTRA}" ]; then
  echo "Unstaging paths not in PHASE5_FILES:"
  echo "${EXTRA}"
  while IFS= read -r x; do
    [ -n "${x}" ] || continue
    git -C "${REPO_DIR}" restore --staged -- "${x}"
  done <<< "${EXTRA}"
fi
```

IF any path was rejected → tell the user which paths were unstaged and why before continuing. Re-run the diff after unstaging to confirm the index now matches `PHASE5_FILES` exactly (accounting for deletions).

**On a `stack` branch, or when Step 0 skipped ahead because a Phase 5 commit already existed,** run the equivalent check against the branch, not just the index:

```bash
git -C "${REPO_DIR}" diff --name-only "${BASE_BRANCH}...HEAD" > "${WORK_CVE}/branch-files.txt"
comm -23 <(sort "${WORK_CVE}/branch-files.txt") <(sort "${PHASE5_FILES}")
```

IF that reports any path outside `PHASE5_FILES` → stop; do not push or open/update the PR. **Always return `status: failed` (`phase5_files_mismatch`) immediately, for manual follow-up. Never gated by `AUTO_APPROVE` and never prompt** — this is committed history, so it cannot be silently unstaged, and regardless of `AUTO_APPROVE` there is no safe way to continue without a human reviewing the branch directly.

**Commit message** (OpenShift `UPSTREAM:` style). Match the **actual** Phase 5 change, not a canned module bump.

If a dependency bump **and** an upstream fix PR number is known (from the Phase 4 remediation plan):

```
UPSTREAM: <upstream-pr>: Bump <module> to <new> for <CVE_ID>
```

If a dependency bump that is OpenShift-only / fork / carry:

```
UPSTREAM: <carry>: Bump <module> to <new> for <CVE_ID>
```

If a dependency bump with no tracked upstream PR:

```
Bump <module> to <new> for <CVE_ID>
```

If the fix is **not** a version bump (code, config, workaround):

```
Fix <CVE_ID>: <short description of the change>
```

Body: one or two sentences describing what changed and why. Include `Fixes: <SOURCE_TICKET>` in Jira mode; omit that line in direct CVE mode. For a bump, include old → new versions. Do not claim a module bump if Phase 5 did not bump a module.

Commit with sign-off (DCO):

```bash
git -C "${REPO_DIR}" commit --signoff -m "${COMMIT_SUBJECT}" -m "${COMMIT_BODY}"
```

Never use `--no-verify` or skip hooks unless the user explicitly asks.

### 3c. Push

```bash
git -C "${REPO_DIR}" push -u origin "${BRANCH_NAME}"
```

For stack updates after amend only, `git push --force-with-lease` — and only if the user approved amending. Default is a normal push. Amending an existing commit and force-pushing is never authorized by `AUTO_APPROVE`; it always requires an interactive user (moot in practice, since `AUTO_APPROVE=yes` never selects `stack` — see Step 2).

---

## Step 4: Create or Update the GitHub PR

### Title

Match the Phase 5 change. Dependency bump:

```
<CVE_ID>: bump <module-short> to <new> [<SOURCE_TICKET>]
```

Omit `[<SOURCE_TICKET>]` in direct CVE mode. Example: `CVE-2026-33186: bump google.golang.org/grpc to v1.79.3 [OCPBUGS-80452]`

Non-bump fix:

```
<CVE_ID>: <short description> [<SOURCE_TICKET>]
```

### Body

**Jira mode** — every Jira key must be a markdown link. Use a `Fixes:` line (do not use a bare key). Describe the **actual** change; the bump wording below is only an example.

```markdown
## Summary

Fixes: [OCPBUGS-80452](https://redhat.atlassian.net/browse/OCPBUGS-80452)

<What Phase 5 changed and why — e.g. bump `<module>` from `<old>` to `<new>`, or a code/config workaround.>

<2–3 sentence CVE description from Phase 1>

## Changes

- <file- or behavior-level bullets from the Phase 5 diff>

## Verification

- `make verify` / `go mod verify` (if modules changed)
- `make build` / `go build ./...`
- `govulncheck ./...`
```

**Direct CVE mode** — same body **without** the `Fixes:` line and without Jira URLs.

Do not include hostnames, cluster names, routes, usernames, passwords, or tokens in the title, body, or comments.

### Create vs update

**New PR:**

```bash
gh pr create \
  --repo "${ORG}/${REPO}" \
  --base "${BASE_BRANCH}" \
  --head "${BRANCH_NAME}" \
  --title "${PR_TITLE}" \
  --body "${PR_BODY}"
```

**Stacked on existing PR:** push is enough. Optionally:

```bash
gh pr comment "${EXISTING_PR}" --repo "${ORG}/${REPO}" \
  --body "Added <CVE_ID> fix: <what Phase 5 changed>."
```

And `gh pr edit` to append the CVE / Jira key to title and the `Fixes:` line if missing.

Capture `PR_URL` from `gh pr create` output or `gh pr view --json url`.

IF create fails (permissions, fork needed) → show the exact `gh` error (no secrets), give the user the title/body to paste, return `status: failed` (`pr_create_failed`). Do not retry more than once.

---

## Step 5: Post PR URL to Jira (Jira mode only)

**Skip this step** if `SOURCE_TICKET` is unset (direct CVE mode) → still `status: success` for the GitHub PR.

This is a **new follow-up comment**. Do **not** edit or replace the Phase 4 analysis comment. Do **not** add/remove labels here.

Use the posting procedure in [report-to-jira](../report-to-jira/SKILL.md) **Follow-up: PR URL comment** (REST API with Internal visibility first, then MCP/CLI fallback with the same public-visibility warning).

Comment body (convert to Jira wiki markup per `report-to-jira` when using the REST API):

```markdown
### Remediation PR opened

A GitHub pull request is open for this CVE.

- **PR:** [PR_URL](PR_URL)
- **CVE:** CVE_ID
- **Change:** <what Phase 5 changed — e.g. `module` old → new, or a short code/config summary>
- **Base branch:** BASE_BRANCH
```

IF posting fails → display the comment in session for manual paste. The GitHub PR is still a success (`jira_followup: failed`).

---

## Return Value

**Success:**

```json
{
  "skill": "create-fix-pr",
  "status": "success",
  "action": "created | updated",
  "pr_url": "https://github.com/<org>/<repo>/pull/<n>",
  "pr_number": 123,
  "branch": "fix/cve-yyyy-nnnnn-ocpbugs-12345",
  "base_branch": "release-4.21",
  "cve_id": "CVE-YYYY-NNNNN",
  "source_ticket": "OCPBUGS-12345 or null",
  "jira_followup": "posted | skipped | failed"
}
```

**Skipped:**

```json
{
  "skill": "create-fix-pr",
  "status": "skipped",
  "reason": "user_declined | no_local_changes | phase5_incomplete | user_wait_for_pr | user_wait_for_pr_auto | embargo"
}
```

**Failed:**

```json
{
  "skill": "create-fix-pr",
  "status": "failed",
  "reason": "git_missing | gh_missing | gh_unauthenticated | pr_create_failed | commit_failed | phase5_files_missing | phase5_files_mismatch",
  "error": "<message without secrets>"
}
```

---

## Integration with Parent Workflow

Called from **Phase 6** of `CLAUDE.md` after Phase 5 verification succeeds.

---

## Guardrails

- Never commit, push, or open a PR without explicit approval — interactive, or `AUTO_APPROVE=yes` recorded once upfront
- Never force-push to `BASE_BRANCH` / `main` / `master` / `release-*`
- Never skip git hooks unless the user asks
- Never stage paths outside `PHASE5_FILES`; never `git add -A` or `git add .`
- Never commit/push/PR without validating the staged (and, for `stack`, the branch) path set against `PHASE5_FILES` and rejecting extras
- Never run vendor sync in Phase 6 — it happens in Phase 5, before `PHASE5_FILES` is written
- Never let `AUTO_APPROVE=yes` auto-select `stack` or `independent` on a PR-title conflict — always `wait` unattended
- Never let `AUTO_APPROVE=yes` skip a hard-fail case (missing/mismatched `PHASE5_FILES`, ambiguous conflict resolution) — those always require a human, flag or not
- Never include secrets in commit messages, PR text, or Jira comments
- Stop immediately on embargo
- Direct CVE mode must still produce a PR
