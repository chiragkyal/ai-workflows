# ai-workflows

[Ambient Code](https://ambient-code.github.io/platform/workflows/custom/) workflow repository. Each subdirectory is a self-contained workflow that can be loaded directly into an Ambient session.

Workflows can also be installed on **[Chai Bot](https://github.com/openshift-eng/ship-help-bot)** Remote Workspace (RWS) pods as Claude Code plugins. Persona and profile wiring lives in ship-help-bot; per-workflow RWS notes live under each workflow's `reference/chai-bot-rws.md` (see [analyze-cve](analyze-cve/reference/chai-bot-rws.md)).

## Structure

```text
{workflow-name}/
├── .ambient/
│   └── ambient.json     # Required for Ambient: name, description, system prompt, startup prompt
├── CLAUDE.md            # Main workflow instructions (phases, decision trees, output formats)
├── commands/            # Slash commands for Chai Bot `/name` entry (optional for Ambient)
│   └── {name}.md
├── skills/              # Reusable sub-agent skills referenced from CLAUDE.md
│   └── {skill-name}/
│       └── SKILL.md
└── reference/           # Runtime-specific notes (e.g. chai-bot-rws.md for RWS)
```

## Loading a Workflow (Ambient Code)

1. Open the **New Session** dialog in Ambient Code.
2. In the **Workflow** dropdown, select **Custom Workflow…**.
3. Enter the Git URL of this repository, the branch, and the path to the workflow directory (e.g. `analyze-cve`).

## Available Workflows

| Workflow | Description |
|----------|-------------|
| [analyze-cve](analyze-cve/) | Analyze a Go codebase for CVE vulnerabilities, assess impact via call graph analysis, generate a remediation plan, and optionally open a GitHub PR after a verified fix. Supports `--auto-approve=yes` for unattended/scheduled runs. |
