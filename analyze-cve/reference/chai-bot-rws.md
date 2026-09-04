# Chai Bot (Remote Workspace) Integration

This workflow runs on [Chai Bot](https://github.com/openshift-eng/ship-help-bot)
Remote Workspace (RWS) pods when installed via the `opae_cve` plugin profile or an
inline `rws.plugins:` URL pointing at this directory.

## Plugin install layout

The coordinator calls the pod manager's `/prepare-plugins`, which writes:

| Artifact | Path on worker |
|----------|----------------|
| Skills | `/workspace/.claude/skills/<skill-name>/SKILL.md` |
| Commands | `/workspace/.claude/commands/analyze-cve.md` |

The worker (Claude Code) discovers these natively. The **coordinator** invokes the
workflow with `rws_query` — e.g. "Run `/analyze-cve --jira=OCPBUGS-12345`" — and
does not paste skill bodies into the prompt.

## Workspace paths (RWS defaults)

Ambient Code uses `/workspace/workflows/ai-workflows` as its persistent mount.
On Chai Bot RWS pods, use `/workspace` instead:

```bash
export AI_WORKFLOWS_WORKSPACE="${AI_WORKFLOWS_WORKSPACE:-/workspace}"
export REPOS_BASE="${AI_WORKFLOWS_WORKSPACE}/.work/repos"
export WORK_CVE="${AI_WORKFLOWS_WORKSPACE}/.work/compliance/analyze-cve/${CVE_ID}"
```

Reports: `${AI_WORKFLOWS_WORKSPACE}/.work/compliance/analyze-cve/{CVE-ID}/report.md`

Pre-cloned repos: check `${REPOS_BASE}` first, then `/workspace/repos` (ephemeral).

## Jira integration (coordinator vs worker)

RWS worker containers do **not** receive `JIRA_API_TOKEN`, `JIRA_EMAIL`, or
Atlassian MCP env vars. Git operations use the manager's transparent git proxy;
Jira read/write uses **coordinator** tools.

### Coordinator responsibilities (Chai Bot persona)

| Workflow step | Coordinator tool (examples) |
|---------------|----------------------------|
| JQL search (Phase 0.3) | `query_jira(jql, max_results=10)` |
| Fetch ticket (Phase 0.5) | `get_jira_issue(issue_key)` |
| Post analysis comment (Phase 4) | `comment_on_jira_issue(...)` with Internal visibility |
| Add `ai-cve-analyzed` label | `priv_jira_update_issue` or label tool after approval |
| Post PR URL (Phase 6) | `comment_on_jira_issue(...)` |

Pass extracted `CVE_ID`, `IMAGE_NAME`, `BRANCH`, and `jira_context` into the
worker via `rws_query` when the coordinator prefetched the ticket.

### Worker responsibilities

- Clone repos, run `govulncheck`, call-graph analysis, apply fixes, `gh pr create`
- If Jira REST/MCP is unavailable in the worker session, **return structured results**
  to the coordinator (report markdown, risk level, PR URL) and let the coordinator
  post to Jira

### Mapping Ambient MCP names → Chai Bot

| Ambient / reference doc | Chai Bot coordinator |
|-------------------------|----------------------|
| `mcp__atlassian__jira_get_issue` | `get_jira_issue(issue_key)` |
| `mcp__atlassian__jira_search_issues` | `query_jira(jql, max_results)` |
| `mcp__atlassian__jira_add_comment` | `comment_on_jira_issue(issue_key, body)` |
| `mcp__atlassian__jira_update_issue` | `priv_jira_update_issue(...)` (may need approval) |

When running **inside the worker** and coordinator Jira tools are exposed via
coordinator MCP, use those tool names instead of `mcp__atlassian__*`.

## Privileged operations (PR creation)

Opening GitHub PRs from the coordinator requires the approval flow:

1. `check_proposal` for `scm_create_change_request`
2. `make_proposal` + `send_response` if pending
3. After approval: `priv_scm_create_change_request`

The worker may create commits locally; the coordinator opens the PR via
`priv_scm_create_change_request` when policy requires it. Follow the persona's
`policy_grants` and instructions.

## Scheduled / headless runs

Use `--auto-approve=yes` for scheduled tasks. The coordinator must end with
`send_response(mode="report")` (scheduled) or post results to Slack.

Persona and plugin profile wiring (`rws_plugin_profiles`, `opae_cve`, etc.) is
documented in [ship-help-bot](https://github.com/openshift-eng/ship-help-bot) —
not in this workflow repo.
