# Veracode Agents

GitHub Copilot agents for Veracode security scanning workflows. Agents go beyond skills — they can make autonomous decisions, spawn subagents, and run multi-step loops without manual prompting at each step.

All agents require the [Veracode MCP server](https://github.com/dipsylala/veracode-mcp) to be installed and configured in VS Code.

## Prerequisites

- The [Veracode MCP server](https://github.com/dipsylala/veracode-mcp) must be installed and configured in VS Code
- An authenticated Veracode account with API credentials configured
- GitHub Copilot with agent mode enabled in VS Code

## Available Agents

| Agent | User-invocable | What it does |
| --- | --- | --- |
| **Veracode Autofix** | Yes | Autonomous scan → triage → fix loop; packages, scans, triages, and applies fixes until all high/critical findings are resolved |
| **Veracode Triage** | No (machine callout) | Returns a machine-readable JSON ranking of all findings by remediation priority; consumed by Veracode Autofix |

## Installation

### Install for a project (team-shared, checked in to source control)

Copy the agent files into your project's `.github/agents/` directory:

```text
<your-project>/.github/agents/veracode-autofix.agent.md
<your-project>/.github/agents/veracode-triage.agent.md
```

### Install for personal use across all projects

*Windows:*

```powershell
Copy-Item *.agent.md "$env:APPDATA\Code\User\agents\"
```

*macOS:*

```bash
cp *.agent.md "$HOME/Library/Application Support/Code/User/agents/"
```

*Linux:*

```bash
cp *.agent.md "$HOME/.config/Code/User/agents/"
```

After placing the files, reload VS Code (`Developer: Reload Window`). User-invocable agents appear in the Copilot Chat mode selector.

## Usage

### Veracode Autofix

Select **Veracode Autofix** from the Copilot Chat mode selector:

```text
Fix all high and critical vulnerabilities in /path/to/my/project
```

The agent runs a full scan → triage → fix loop autonomously, rescanning after every 10 changed files to confirm fixes. It reports findings fixed, mitigated by design, and any that remain at the end of the run.

### Veracode Triage

Not directly invocable by users. Called automatically by **Veracode Autofix** during Phase 2 of its loop to retrieve a machine-readable JSON ranking of findings.

## Agentic Remediation Loop

**Veracode Autofix** and **Veracode Triage** together implement an autonomous scan-triage-fix loop:

```text
┌─────────────────────────────────────────────────────────────┐
│  Phase 1 — Scan                                             │
│    package-workspace                                        │
│    pipeline-scan (synchronous: true)                        │
│    local-sca-scan                                           │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  Phase 2 — Triage                                           │
│    Spawn "Veracode Triage" → JSON findings array            │
│    No severity ≥ 4? → STOP (clean)                         │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  Phase 3 — Fix (iterate by remediationPriority)             │
│    SAST: remediation-guidance → apply smallest safe fix     │
│    SCA:  identify safe version → upgrade via package mgr    │
│    After each fix: run build → auto-fix compile errors      │
│    Track changed files                                      │
│    > 10 files changed? → Phase 4                           │
└────────────────────────┬────────────────────────────────────┘
                         │ (all attempted or no more sev ≥ 4)
┌────────────────────────▼────────────────────────────────────┐
│  Final Report                                               │
│    Findings fixed / remaining / files changed / scans run   │
└─────────────────────────────────────────────────────────────┘

Phase 4 (10-file checkpoint):
  → Re-run Phase 1 (rescan)
  → Re-run Phase 2 (re-triage)
  → Reset changed-file counter
  → No severity ≥ 4? STOP (clean). Otherwise back to Phase 3.
```

### Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Triage output format | JSON array | Machine-readable; consumed by Autofix without parsing prose |
| Scan mode | `synchronous: true` | Autofix must wait for results before triaging |
| File-change gate | > 10 files | Balances thoroughness against scan turnaround time |
| Post-gate behaviour | Continue looping | Loop runs until all high/critical are resolved |
| Compilation errors | Auto-fix | Prevents compounding errors across subsequent fixes |
| Severity threshold | ≥ 4 (High + VeryHigh) | Focus on exploitable, high-impact issues first |
| Autofix user-invocable | `true` | Accessible directly from the VS Code agent picker |
| Triage user-invocable | `false` | Intended as a machine callout, not a user-facing agent |

### Triage JSON Schema

```json
[
  {
    "id": "<pipeline flaw ID or CVE/SID>",
    "type": "sast | sca",
    "severity": 0,
    "cwe": 89,
    "title": "<flaw title>",
    "file": "<workspace-relative path or null>",
    "line": 42,
    "component": "<name@version for SCA, null for SAST>",
    "cve": "<CVE/SID for SCA, null for SAST>",
    "remediationPriority": 1,
    "remediationRationale": "<one sentence>",
    "mitigationCandidate": false
  }
]
```

**Ranking rules:**

- Severity first (5 = VeryHigh → 0 = Informational)
- SCA findings with CVSS ≥ 9.0 are bumped to VeryHigh regardless of Veracode severity
- Among equal severities: exploitability context first, then ease of fix (quick wins rank higher)

## Agents vs Skills

[Skills](https://github.com/dipsylala/veracode-skills) are lightweight prompt instructions that guide Copilot to call the right MCP tools for a specific task. They are ideal for interactive, single-purpose workflows.

Agents are more capable: they can make autonomous decisions, spawn subagents, maintain loop state, and chain multi-step operations. Use agents when you want hands-off automation rather than guided assistance.


## See Also

- [veracode-skills](https://github.com/dipsylala/veracode-skills) — Lightweight task-focused Copilot skills for Veracode workflows
- [veracode-mcp](https://github.com/dipsylala/veracode-mcp) — The MCP server these agents connect to
