# ai-workflows

[Ambient Code](https://ambient-code.github.io/platform/workflows/custom/) workflow repository. Each subdirectory is a self-contained workflow that can be loaded directly into an Ambient session.

## Structure

```text
{workflow-name}/
├── .ambient/
│   └── ambient.json     # Required: workflow name, description, system prompt, startup prompt
├── CLAUDE.md            # Main workflow instructions (phases, decision trees, output formats)
└── skills/              # Optional: reusable sub-agent skills referenced from CLAUDE.md
    └── {skill-name}/
        └── SKILL.md
```

## Loading a Workflow

1. Open the **New Session** dialog in Ambient Code.
2. In the **Workflow** dropdown, select **Custom Workflow…**.
3. Enter the Git URL of this repository, the branch, and the path to the workflow directory (e.g. `analyze-cve`).

## Available Workflows

| Workflow | Description |
|----------|-------------|
| [analyze-cve](analyze-cve/) | Analyze a Go codebase for CVE vulnerabilities, assess impact via call graph analysis, generate a remediation plan, and optionally open a GitHub PR after a verified fix. Supports `--auto-approve=yes` for unattended/scheduled runs. |
