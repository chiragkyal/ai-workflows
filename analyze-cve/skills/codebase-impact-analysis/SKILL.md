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

```bash
# Run with a hard timeout — govulncheck downloads the vuln DB and scans
# all packages; on large repos or slow connections it can run indefinitely.
timeout 300 govulncheck ./... 2>&1
EXIT_CODE=$?

if [ $EXIT_CODE -eq 124 ]; then
  echo "govulncheck timed out after 300s — skipping scanner, continuing with manual methods"
fi
```

**Timeout handling:**
- `timeout 300` kills the process after 5 minutes and returns exit code 124
- IF timed out → note "govulncheck skipped (timeout)" in evidence, continue to Method 3
- Do NOT block the analysis on govulncheck — it is one signal, not the only one

- Parse output for CVE matches
- Check if vulnerable symbols are reported as called

**Decision Point:**
- IF govulncheck confirms vulnerability → Strong evidence for HIGH RISK
- IF govulncheck does not find it → Continue to Method 3 (CVE may not be in Go vulndb yet)
- IF govulncheck timed out → Continue to Method 3; note the gap in the report

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
