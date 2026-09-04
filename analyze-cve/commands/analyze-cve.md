---
description: Run the full Go CVE analysis workflow — intelligence gathering, call-graph impact analysis, remediation, optional fix application and GitHub PR
argument-hint: "[CVE-YYYY-NNNNN | --jira=KEY | --jql=\"...\"] [--repo=...] [--algo=vta] [--auto-approve=yes|no]"
---

# analyze-cve

Entry point for the CVE Analysis workflow. Follow **every phase** in `CLAUDE.md`
in the plugin root (parent directory of this `commands/` folder).

## Usage examples

```
/analyze-cve CVE-2024-45338 --repo=https://github.com/openshift/cert-manager-operator
/analyze-cve --jira=OCPBUGS-12345
/analyze-cve --jira=OCPBUGS-12345 --repo=cert-manager-operator --auto-approve=no
/analyze-cve --jql="project = OCPBUGS AND labels = needs-cve-analysis ORDER BY created ASC"
```

## Runtime notes

- **Chai Bot (RWS):** See `reference/chai-bot-rws.md` for workspace paths and Jira
  tool mapping. Default workspace root is `/workspace` (not Ambient's workflow mount).
- **Sub-skills** under `skills/` are invoked by phase; do not skip phases unless
  `CLAUDE.md` explicitly allows it.
- **Credentials:** Never log tokens or API keys. Use env var references only.

Start with Phase 0 (argument parsing and tool validation), then proceed in order.
