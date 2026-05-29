---
name: codebase-impact-analysis
description: Analyze a Go codebase to determine if it is impacted by a specific CVE using multiple verification methods and assign a risk level
---

# Codebase Impact Analysis

Determines whether a Go codebase is impacted by a specific CVE by applying multiple analysis methods with increasing confidence, collecting evidence, and assigning a risk level.

## When to Use This Skill

Use this skill when:
- A CVE profile has been gathered (from the cve-intelligence-gathering skill)
- You need to determine if the current Go project is affected
- You need to assign a risk level with supporting evidence

## Prerequisites

### Required Tools (validated in Phase 0 of parent command)
- `go` toolchain with `go.mod` in workspace root
- `govulncheck`: `go install golang.org/x/vuln/cmd/govulncheck@latest`
- `callgraph`: `go install golang.org/x/tools/cmd/callgraph@latest`
- `digraph`: `go install golang.org/x/tools/cmd/digraph@latest`

### Required Inputs

**From Phase 1 (cve-intelligence-gathering skill):**
- CVE ID
- Affected package/module name(s)
- Vulnerable version range
- Fixed version (if available)
- Vulnerable function signatures (if known)

**From Parent Command:**
- `--algo` preference for call graph analysis (default: `vta`)

## Implementation Steps

### Step 1: Identify Go Module Dependencies

```bash
# Parse dependencies from go.mod
go list -m all

# Get detailed dependency info
go list -m -json all
```

- Read `go.mod` from workspace root
- Parse direct and indirect dependencies
- Extract module versions

### Step 2: Cross-Reference Vulnerable Packages

Apply the following methods in order. Each provides increasing confidence.

#### Method 1: Dependency Matching

- Compare CVE-affected packages with `go.mod` dependencies
- Check if affected package versions are in use
- Account for version ranges and semantic versioning

```bash
# Check if vulnerable package is a dependency
go list -m <vulnerable-package>
```

**Decision Point:**
- IF package NOT in dependencies → Skip to risk assignment (likely LOW RISK)
- IF package found → Continue to Method 2

#### Method 2: Go Vulnerability Scanner

> **CRITICAL RULES — read before running anything:**
> 1. **Run govulncheck AT MOST ONCE per phase.** If `/tmp/govulncheck-module.txt` or `/tmp/govulncheck-source.txt` already exist and are non-empty, use those files. Do NOT re-run govulncheck.
> 2. **Never pipe govulncheck to `head`, `tail`, or any other command.** Always redirect to a file. Piping causes govulncheck to hang (SIGPIPE) when the reader closes.
> 3. **"No findings" is a valid and final result** — it means the CVE is not yet in the Go vuln database. Immediately proceed to Method 3. Do NOT re-run in a different mode or format to double-check.

**Phase 2a — Module-level scan (always run first, fast)**

Checks go.mod only. Completes in seconds regardless of repo size.

```bash
# Only run if output file does not already exist
if [ ! -s /tmp/govulncheck-module.txt ]; then
  echo "=== govulncheck module scan ==="
  timeout 60 govulncheck -scan=module ./... > /tmp/govulncheck-module.txt 2>&1
  MODULE_EXIT=$?
  if [ $MODULE_EXIT -eq 124 ]; then
    echo "govulncheck module scan timed out — skipping scanner entirely"
  fi
else
  echo "=== govulncheck module scan (using cached result) ==="
fi
cat /tmp/govulncheck-module.txt
```

- IF the vulnerable package appears in output → proceed to Phase 2b
- IF the vulnerable package is absent → record "package not in module graph" as LOW signal; **skip Phase 2b immediately**; proceed to Method 3
- IF timed out → skip Phase 2b; note gap; proceed to Method 3

**Phase 2b — Source-level scan (only if Phase 2a confirms package is present)**

Full symbol-level analysis. Can be slow on large repos (e.g. spiffe-spire, cert-manager).

```bash
# Only run if output file does not already exist
if [ ! -s /tmp/govulncheck-source.txt ]; then
  echo "=== govulncheck source scan ==="
  timeout 300 govulncheck ./... > /tmp/govulncheck-source.txt 2>&1
  SOURCE_EXIT=$?
  if [ $SOURCE_EXIT -eq 124 ]; then
    echo "govulncheck source scan timed out after 300s — relying on manual methods"
  fi
else
  echo "=== govulncheck source scan (using cached result) ==="
fi
cat /tmp/govulncheck-source.txt
```

- IF timed out (exit 124) → note "govulncheck source scan skipped (timeout)" in evidence; **proceed to Method 3 immediately**
- Save `/tmp/govulncheck-source.txt` as a workflow artifact

**Decision Point — govulncheck is ONE signal, not the only one. Always proceed to Method 3 next.**
- IF source scan reports vulnerable symbols called → Strong evidence for HIGH RISK; still continue to Method 3
- IF source scan reports **no findings** → The CVE is likely not yet in the Go vuln database. This is conclusive for govulncheck. **Do NOT re-run in a different format or mode.** Proceed to Method 3.
- IF source scan timed out or was skipped → Proceed to Method 3; note the gap in the report

#### Method 3: Direct Dependency Check

```bash
# Verify package is included (directly or transitively)
go list -mod=mod <vulnerable-package>
```

**Note:** Package presence alone doesn't prove vulnerable functions are called.

#### Method 4: Source Code Analysis

- Search for import statements of vulnerable packages in source code
- Use grep/codebase_search to find package usage
- Search for vulnerable function/method names in codebase
- Identify actual code paths that use vulnerable functions
- Check if vulnerable functions are called in reachable code

#### Method 5: Call Graph Reachability Analysis (Mandatory when package is present)

Delegate to the [call-graph-analysis](../call-graph-analysis/SKILL.md) skill.

- **Pass**: `--algo` preference from user, vulnerable function signature, package path
- **Receive**: Risk level, call chain, evidence files

**This method is REQUIRED whenever the vulnerable package is present in `go.mod`** — regardless of what Methods 2, 3, or 4 found. Source code analysis (Method 4) is heuristic: it can miss indirect calls through interfaces, generated code, and runtime dispatch. Only a call graph provides provable reachability.

**Valid reasons to skip call graph:**
- Package is NOT in `go.mod` (genuinely unreachable — LOW RISK by definition)
- Codebase does not compile (note the limitation; rely on other methods)
- Vulnerable function signature is unknown (note the gap; rely on govulncheck and source analysis)

**NOT a valid reason to skip:**
- Source code analysis found no direct calls to the vulnerable function
- govulncheck did not flag it (CVE may not be in the Go vuln DB yet)
- The analysis "feels" complete from earlier methods

#### Method 6: Configuration and Context Analysis

- Review if vulnerable features are actually enabled
- Check if vulnerable code paths are behind feature flags
- Verify if inputs can reach vulnerable functions
- Consider security controls (input validation, sandboxing)

### Confidence Levels

Each method provides increasing confidence:

1. **Basic Presence** (Low) — Package in `go.mod` (Method 1, 3)
2. **Import & Version Analysis** (Medium) — Package imported, version in vulnerable range, function names found (Method 4)
3. **Vulnerability Scanner** (Medium-High) — `govulncheck` confirms reachable vulnerable symbols (Method 2)
4. **Call Graph Reachability** (Definitive) — Proven execution path from entry point to vulnerable function (Method 5)
5. **Context Analysis** — Mitigating or aggravating factors (Method 6)

**Required minimum:** If the package is in `go.mod`, the analysis is not complete until Method 5 has run or a valid skip reason has been documented. Never stop at Method 4 alone.

### Step 3: Build Evidence Package

Collect evidence from all methods used:

- **Dependency Evidence**: `go.mod` entries, `go list` output, version info
- **Static Code Evidence**: File paths, line numbers, code snippets showing usage
- **Reachability Evidence**: Call graph output, execution paths, DOT visualization (saved to `.work/compliance/analyze-cve/{CVE-ID}/callgraph.svg`)
- **Scanner Evidence**: `govulncheck` output, vulnerability findings
- **Mitigation Factors**: Input validation, disabled features, feature flags, security controls

### Step 4: Assign Risk Level

Evaluate all evidence and assign a risk level. The determination should be data-driven, not formula-based.

**HIGH RISK:**
- Call graph shows reachable path to vulnerable function, OR govulncheck confirms vulnerability

**MEDIUM RISK:**
- Package + vulnerable version in dependencies, usage evidence present, but call graph could not run (build failure or unknown function signature) — reachability not definitively proven

**LOW RISK:**
- Package not in dependencies (call graph skipped — genuinely unreachable), OR version not in vulnerable range, OR call graph ran and found no reachable path

**NEEDS REVIEW:**
- Package is in `go.mod` but call graph was skipped for any reason other than package absence or build failure — escalate; do not leave as LOW based on source code analysis alone
- Conflicting signals or incomplete analysis

> **Rule:** If package is in `go.mod` and call graph was skipped because source analysis "found nothing", assign **NEEDS REVIEW**, not LOW RISK. Document the skip reason explicitly.

## Return Value

Return structured result to parent command:

```json
{
  "skill": "codebase-impact-analysis",
  "status": "success",
  "risk_level": "<HIGH|MEDIUM|LOW|NEEDS_REVIEW>",
  "methods_used": ["dependency_matching", "govulncheck", "direct_dependency_check", "source_code_analysis", "call_graph", "context_analysis"],
  "evidence": {
    "dependency": {
      "package_found": true,
      "current_version": "<version>",
      "dependency_type": "<direct|indirect>",
      "in_vulnerable_range": true
    },
    "govulncheck": {
      "ran": true,
      "cve_found": true,
      "vulnerable_symbols_called": true
    },
    "call_graph": {
      "ran": true,
      "algorithm": "<vta|rta|cha|static>",
      "reachable_from_main": true,
      "call_chain": "main -> handler -> parse -> VULN",
      "evidence_files": ["callgraph.dot", "callgraph.svg"],
      "skip_reason": "<null if ran | 'package_not_in_gomod' | 'build_failure' | 'unknown_function_signature'>"
    },
    "source_analysis": {
      "import_found": true,
      "function_usage_found": true,
      "files": ["<file1>:<line>", "<file2>:<line>"]
    },
    "mitigation_factors": []
  },
  "confidence_assessment": {
    "level": "<HIGH|MEDIUM|LOW>",
    "methods_count": 4,
    "gaps": ["<any gaps in analysis>"]
  }
}
```

## Error Handling

### Build Failures
- IF project doesn't compile → Note limitation, skip call graph analysis, rely on other methods

### Missing CVE in govulncheck Database
- IF govulncheck doesn't know about this CVE → Continue with other methods, note gap

### Large Codebases
- IF call graph times out → Follow fallback strategy in call-graph-analysis skill (algorithm fallback, then scope narrowing)

### Incomplete CVE Information
- IF vulnerable function signature unknown → Skip call graph, note `skip_reason: unknown_function_signature`, assign MEDIUM at best — do NOT assign LOW based on source analysis alone

### Source Code Analysis Shows No Usage
- This is **NOT** a valid reason to skip call graph. Proceed with Method 5.
- Source code search misses interface dispatch, generated code, and indirect call paths.

## Integration with Parent Command

This skill is called from Phase 2 of the CVE Analysis workflow (see `CLAUDE.md`).

**Input:** CVE profile from Phase 1, `--algo` preference from user
**Output:** Risk level, evidence package, confidence assessment
**Next:** Parent command uses risk level to decide whether to generate report and proceed to remediation
