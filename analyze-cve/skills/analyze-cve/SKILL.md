---
name: analyze-cve
description: Full Go CVE analysis workflow — Jira/CVE intake, govulncheck and call-graph impact analysis, remediation planning, optional fix application and GitHub PR
---

# analyze-cve (orchestrator)

Run the complete CVE analysis pipeline defined in the plugin root `CLAUDE.md`.

## When to use

- User provides a CVE ID, Jira ticket (`--jira=`), or JQL batch (`--jql=`)
- OpenShift / Go component impact analysis with optional automated fix + PR

## How to run

1. Read and follow `CLAUDE.md` from the plugin root (all phases 0–6).
2. Invoke phase-specific sub-skills under `skills/` as directed by `CLAUDE.md`.
3. On **Chai Bot Remote Workspace**, read `reference/chai-bot-rws.md` first for
   path defaults and Jira integration (coordinator vs worker).

## Arguments (pass through from user)

| Input | Example |
|-------|---------|
| CVE ID | `CVE-2024-45338` |
| Jira ticket | `--jira=OCPBUGS-12345` |
| JQL batch | `--jql="project = OCPBUGS AND ..."` |
| Repository | `--repo=https://github.com/openshift/hypershift` or `--repo=hypershift` |
| Call-graph algo | `--algo=vta` (default), `rta`, `cha`, `static` |
| Autonomous mode | `--auto-approve=yes` (scheduled) or `no` (interactive, default) |

## Sub-skills (by phase)

| Phase | Skill |
|-------|-------|
| 0.5 | `jira-cve-extraction` |
| 0.7 | `image-repo-mapping` |
| 1 | `cve-intelligence-gathering` |
| 2 | `codebase-impact-analysis` (+ `call-graph-analysis`) |
| 4 | `remediation-planning`, `report-to-jira` |
| 6 | `create-fix-pr` |

Do not paraphrase skill logic — load and execute each skill's `SKILL.md` when its phase runs.
