---
name: report-to-jira
description: Post the final CVE analysis report as a comment on the OAPE-751 Jira ticket, prefixed with an AI workflow attribution header
---

# Report to Jira

Posts the completed CVE analysis report as a comment on **OAPE-751** in the Red Hat Jira instance (`redhat.atlassian.net`). Always called as the last step of Phase 4, after the report has been generated.

---

## Step 1: Confirm Report is Ready

Before posting, verify that the following are available from the parent workflow:

- Final report content (full markdown from Phase 4)
- CVE ID (e.g. `CVE-2024-45338`)
- Source Jira ticket key, if analysis was triggered via `--jira=` (e.g. `OCPBUGS-12345`)
- Risk level (`HIGH` / `MEDIUM` / `LOW` / `NEEDS_REVIEW`)
- Repository analysed (e.g. `https://github.com/openshift/spire-operator`)

If the report is incomplete or Phase 4 did not finish, do not post — return `status: skipped` with reason.

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

**Primary — MCP tool:**

```python
mcp__atlassian__jira_add_issue_comment(
    issue_key="OAPE-751",
    comment_body="<constructed comment from Step 2>"
)
```

**Fallback — jira-cli** (if MCP tool is unavailable):

```bash
jira issue comment add OAPE-751 \
  --body "$(cat /tmp/cve-report-comment.txt)" \
  --no-input
```

Write the comment body to `/tmp/cve-report-comment.txt` before running the fallback.

---

## Step 4: Confirm and Report

After posting, output to the session:

```
✅ Report posted to OAPE-751
   https://redhat.atlassian.net/browse/OAPE-751

   CVE:        <CVE_ID>
   Risk level: <level>
   Repository: <repo_url>
   Session:    <SESSION_URL or "(not available)">
```

If the post fails:

```
❌ Failed to post report to OAPE-751
   Error: <error message>

   The full report has been displayed above. Please copy and paste it
   into OAPE-751 manually:
   https://redhat.atlassian.net/browse/OAPE-751
```

Do not retry more than once. On failure, display the comment body in the session so the user can post it manually.

---

## Return Value

**Success:**
```json
{
  "skill": "report-to-jira",
  "status": "success",
  "target_ticket": "OAPE-751",
  "ticket_url": "https://redhat.atlassian.net/browse/OAPE-751",
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
  "reason": "<report incomplete | phase 4 did not finish>"
}
```

**Failed:**
```json
{
  "skill": "report-to-jira",
  "status": "failed",
  "target_ticket": "OAPE-751",
  "error": "<error message>",
  "fallback": "comment body displayed in session for manual posting"
}
```

---

## Integration with Parent Workflow

Called from **Phase 4** of the CVE Analysis workflow (see `CLAUDE.md`) as the final step, after the report has been fully generated.

**Input:** complete report content, CVE ID, risk level, repo URL, optional source Jira ticket key  
**Output:** confirmation of comment posted to OAPE-751, or failure message with comment body for manual posting
