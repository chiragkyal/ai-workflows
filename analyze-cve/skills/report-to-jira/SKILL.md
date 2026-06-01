---
name: report-to-jira
description: Post the final CVE analysis report as a comment on the source Jira ticket, prefixed with an AI workflow attribution header
---

# Report to Jira

Posts the completed CVE analysis report as a comment on the **source Jira ticket** — the same ticket the CVE details were read from (passed via `--jira=`, e.g. `OCPBUGS-12345`). The comment and label always go back to this one ticket. Always called as the last step of Phase 4, after the report has been generated.

---

## Step 1: Confirm Report is Ready

Before posting, verify that the following are available from the parent workflow:

- Final report content (full markdown from Phase 4)
- CVE ID (e.g. `CVE-2024-45338`)
- **`SOURCE_TICKET`** — the Jira ticket key from `--jira=` (e.g. `OCPBUGS-12345`). This is the only ticket this skill will write to.
- Risk level (`HIGH` / `MEDIUM` / `LOW` / `NEEDS_REVIEW`)
- Repository analysed (e.g. `https://github.com/openshift/spire-operator`)

**If `SOURCE_TICKET` is not available** (direct CVE mode — no `--jira` was provided): return `status: skipped` with reason `"no_source_ticket"`. There is no Jira ticket to post to.

If the report is incomplete or Phase 4 did not finish, return `status: skipped` with reason.

---

## Step 1.5: Resolve the Session URL

Ambient injects the current session URL as an environment variable. Run:

```bash
echo "${AMBIENT_SESSION_URL:-}"
echo "${SESSION_URL:-}"
echo "${AMBIENT_SESSION_ID:-}"
```

Build `SESSION_URL` using the first value found:

| Variable present | Value to use |
|---|---|
| `AMBIENT_SESSION_URL` | Use directly |
| `SESSION_URL` | Use directly |
| `AMBIENT_SESSION_ID` | Construct: `${AMBIENT_BASE_URL}/sessions/${AMBIENT_SESSION_ID}` |
| None found | Leave blank — omit session link from comment gracefully |

Store as `SESSION_URL` (or empty string). This is used in Step 2.

---

## Step 2: Build the Comment Body

Convert the full Phase 3 report to Jira wiki markup and post it in its entirety. Do not summarise — the full report is the comment.

### Conversion rules (markdown → Jira wiki markup)

| Markdown | Jira wiki markup |
|---|---|
| `## Heading` | `h2. Heading` |
| `### Heading` | `h3. Heading` |
| `**bold**` | `*bold*` |
| `_italic_` | `_italic_` |
| `` `inline code` `` | `{{inline code}}` |
| ` ```code block``` ` | `{code}<br>...<br>{code}` |
| `\| table \| row \|` | `\| table \| row \|` (unchanged) |
| `\|\| header \|\|` | `\|\| header \|\|` (unchanged) |
| `- bullet` | `* bullet` |
| `[text](url)` | `[text\|url]` |
| `---` (horizontal rule) | `----` |

Prepend the following attribution header before the converted report:

```
⚠️ *This analysis was performed automatically by the CVE Analysis AI Workflow.*
_Results should be reviewed by a human before acting on remediation steps._
_Workflow: [analyze-cve|https://github.com/chiragkyal/ai-workflows/tree/main/analyze-cve]_
<IF SESSION_URL is non-empty:>
_Session: [View full session transcript|<SESSION_URL>]_
</IF>

----

```

If `SESSION_URL` is empty, omit the `_Session:_` line entirely — do not print a blank line in its place.

### Size limit handling

Jira comment bodies are capped at **32,767 characters**. Measure the full converted comment length before posting:

- **≤ 32,000 chars** → post in full, no changes needed.
- **> 32,000 chars** → trim raw tool output sections only (govulncheck full output, call graph dot/svg content) and replace each with a one-line note:
  ```
  _(govulncheck full output truncated — available in session artifacts)_
  ```
  Retain all findings, risk assessment, executive summary, and remediation sections in full. Re-measure after trimming and repeat if still over limit.

### Stripping rules (apply regardless of size)

- Remove internal email addresses (e.g. `user@redhat.com`) — replace with the display name only.
- Do not include embargoed content — if somehow reached here with `embargo_status: True`, abort immediately.

---

## Step 3: Post the Comment

> **Credential rule:** Never print, echo, or log any token, key, or password value. Pass credentials only via environment variable references (e.g. `$JIRA_TOKEN`). If `jira-cli` or `curl` requires authentication, confirm the call succeeded by checking the exit code — never by echoing the token value.

Post to `SOURCE_TICKET` — the ticket the CVE details were read from.

**Primary — MCP tool:**

```python
mcp__atlassian__jira_add_issue_comment(
    issue_key=SOURCE_TICKET,       # e.g. "OCPBUGS-12345"
    comment_body="<constructed comment from Step 2>"
)
```

**Fallback — jira-cli** (if MCP tool is unavailable):

```bash
jira issue comment add "${SOURCE_TICKET}" \
  --body "$(cat /tmp/cve-report-comment.txt)" \
  --no-input
```

Write the comment body to `/tmp/cve-report-comment.txt` before running the fallback.

---

## Step 4: Confirm and Report

After posting, output to the session:

```
✅ Report posted to <SOURCE_TICKET>
   https://redhat.atlassian.net/browse/<SOURCE_TICKET>

   CVE:        <CVE_ID>
   Risk level: <level>
   Repository: <repo_url>
   Session:    <SESSION_URL or "(not available)">
```

If the post fails:

```
❌ Failed to post report to <SOURCE_TICKET>
   Error: <error message>

   The full report has been displayed above. Please copy and paste it
   into <SOURCE_TICKET> manually:
   https://redhat.atlassian.net/browse/<SOURCE_TICKET>
```

Do not retry more than once. On failure, display the comment body in the session so the user can post it manually.

---

## Step 4.5: Mark Source Ticket as Processed

Add the label **`ai-cve-analyzed`** to `SOURCE_TICKET` to prevent redundant re-processing on future runs.

**Only run this step if Step 3 succeeded.** The label confirms the full workflow completed — comment posted AND analysis done. If Step 3 failed for any reason (MCP error, jira-cli not found, network issue), **skip this step entirely**. Do not add the label to a ticket that did not receive the comment.

**Do NOT run if:**
- Step 3 failed or was skipped for any reason
- `SOURCE_TICKET` is not set (direct CVE mode — no Jira ticket)

### ⚠️ CRITICAL: Existing labels MUST be preserved

Jira's update API **replaces** the entire label list — it does not append. Sending only `["ai-cve-analyzed"]` will **delete all existing labels** on the ticket. This is a destructive operation and must never happen.

**Before writing, always:**
1. Take the full `labels` list already captured in `jira_context["labels"]` (fetched in Phase 0.5 — no extra API call needed)
2. Append `ai-cve-analyzed` to that list
3. Write the combined list back

```python
# 1. Take the labels captured during Phase 0.5 — never start from an empty list
current_labels = jira_context["labels"]   # e.g. ["CVE-2026-34986", "SecurityTracking", "pscomponent:..."]

# 2. Append only if not already present
if "ai-cve-analyzed" not in current_labels:
    new_labels = current_labels + ["ai-cve-analyzed"]
else:
    new_labels = current_labels   # already marked — nothing to write

# 3. Write the FULL list back
mcp__atlassian__jira_update_issue(
    issue_key=SOURCE_TICKET,
    fields={},
    additional_fields={
        "labels": new_labels
    }
)
```

**Fallback — jira-cli** (if MCP tool unavailable):

```bash
# jira-cli --label appends without replacing — safe to use directly
jira issue edit "${SOURCE_TICKET}" --label "ai-cve-analyzed" --no-input
```

### Verification (mandatory)

After the update call, re-fetch the ticket labels and confirm:

```python
updated = mcp__atlassian__jira_get_issue(issue_key=SOURCE_TICKET, fields="labels")
updated_labels = updated["fields"]["labels"]

# Check 1: new label was added
assert "ai-cve-analyzed" in updated_labels, "New label missing"

# Check 2: all original labels are still present
for label in current_labels:
    assert label in updated_labels, f"LABEL LOST: {label}"
```

**If Check 1 fails (new label missing):** log a warning — non-fatal. The report is already posted.

```
⚠️ Could not add 'ai-cve-analyzed' label to <SOURCE_TICKET>. Add it manually to prevent re-processing.
```

**If Check 2 fails (existing label lost):** this is a data integrity error. Log a critical error and output the original label list so the user can restore it:

```
❌ LABEL INTEGRITY ERROR on <SOURCE_TICKET>
   The following labels were present before the update but are now missing:
   <list of lost labels>

   Original full label list (restore manually):
   <current_labels>
```

**If both checks pass:**

```
✅ Label 'ai-cve-analyzed' added to <SOURCE_TICKET>. Labels verified intact.
```

---

## Return Value

**Success:**
```json
{
  "skill": "report-to-jira",
  "status": "success",
  "source_ticket": "<SOURCE_TICKET e.g. OCPBUGS-12345>",
  "ticket_url": "https://redhat.atlassian.net/browse/<SOURCE_TICKET>",
  "cve_id": "<CVE_ID>",
  "risk_level": "<HIGH|MEDIUM|LOW|NEEDS_REVIEW>",
  "method": "mcp | jira-cli",
  "session_url": "<SESSION_URL or null>"
}
```

**Skipped:**
```json
{
  "skill": "report-to-jira",
  "status": "skipped",
  "reason": "<no_source_ticket | report incomplete | phase 4 did not finish>"
}
```

**Failed:**
```json
{
  "skill": "report-to-jira",
  "status": "failed",
  "source_ticket": "<SOURCE_TICKET>",
  "error": "<error message>",
  "fallback": "comment body displayed in session for manual posting"
}
```

---

## Integration with Parent Workflow

Called from **Phase 4** of the CVE Analysis workflow (see `CLAUDE.md`) as the final step, after the report has been fully generated.

**Input:** complete report content, CVE ID, risk level, repo URL, `SOURCE_TICKET` (Jira ticket key from `--jira=` — the same ticket the CVE details were read from)  
**Output:** confirmation of comment and label posted to `SOURCE_TICKET`, or `status: skipped` if no source ticket was provided, or failure message with comment body for manual posting
