---
name: bob-devsecops-package
description: Load when a task mentions IBM Bob Premium Package for DevSecOps (Bob Findings, SafeWrite, /attack-surface-scan, /file-sweep-scan, Trivy/KICS scans) and you must decide whether a Bob Shell worker can use it or how to get security work out of Bob Shell.
---

# Bob Premium Package for DevSecOps, seen from Bob Shell

What this gives the steering agent: the IDE-vs-Shell verdict, what the package offers (for naming and judging claims), and the Shell-side alternative.

## When to load
- A brief or user request says "run the DevSecOps scan", "use SafeWrite", "check Bob Findings", `/attack-surface-scan`, `/file-sweep-scan`, `/terraform-devops`, `/ansible-automation`.
- You plan a security review or dependency fix via `bob run` / `bob chat` and wonder what Bob brings on its own.
- A worker reports scanner findings (Trivy, Opengrep, KICS) and you must judge whether they could be real.

## Verdict: IDE-only, not usable from Bob Shell (as documented)
- All pages are under `/docs/ide/premium-packages/devsecops/`. None says Shell is supported, and no Shell page describes the package (docs). Do not claim Shell support.
- Everything is described as IDE surface: the Bob Findings panel, the Security dashboard, the "Fix with Bob" button, command-palette commands ("Scan for issues", "Run Supply Chain Scan"), a right-click directory scan, SafeWrite as an extension setting (experimental, off by default), and a workflow started from the chat panel (docs).
- Requirements: IBM Bob 2.2.2 or later, active package license (docs). Local Shell is 2.0.6 (verified 2.0.6: `bob --help` shows only `chat`, `run`, `mcp`, `acp`, no extension or package command). The version numbers are not comparable evidence either way; the docs simply give no Shell route.
- One weak hint only: the Shell telemetry page lists a "DevSecOps scans" event. That is a shared telemetry table, not a feature statement.
- Group policies are "supported by Bob IDE only" (docs `ide/security/group-policies`), including the `EnforcedHooks` and `DisabledAutoApprovalGroups` keys; do not assume they bind a Shell worker.
- Practical rule: a Shell worker has none of: continuous scanning, Findings, SafeWrite, the dashboard, the two AI scans, the Terraform and Ansible skills from the package. If it reports them as used, there is no `tool_use` evidence for them; treat as invention.

## What the package offers (IDE, docs) - vocabulary only
| Part | Content |
|---|---|
| Scanners (server side, no install) | Trivy (vulns, secrets, containers), Opengrep with IBM-curated Bob rules (SAST), KICS (IaC misconfig). The extension sends relevant code to the Bob backend |
| SAST languages | JS/TS, Python, Java, Go, C/C++, PHP, Ruby |
| IaC and config | Terraform/OpenTofu, Kubernetes/Helm, Docker, Ansible, CloudFormation/SAM, ARM/Bicep, GitHub Actions and GitLab CI |
| Supply chain | Manifests of npm/yarn/pnpm/bun, pip/Poetry/uv/Pipenv, Maven/Gradle; checked on open and save; risky installs during agent tasks are blocked or replaced by a safe version |
| SafeWrite | Intercepts agent file writes in memory, patches vulnerable dependency declarations, guards lockfiles, optional SAST on proposed code |
| AI scans | `/attack-surface-scan` (top-down, cheaper) and `/file-sweep-scan` (every file, multi-round, more Bobcoins and time) |
| Workflow | Fix Supply Chain Vulnerabilities (runs your test suite at each stage) |
| Skills | `/ansible-automation`, `/terraform-devops` (optional HashiCorp Terraform MCP server) |
| Dashboard | Results per project and branch, kept up to 90 days, HTML export |

## What to do instead from a steering agent
1. Run the scanners yourself where available (for example `trivy fs`, `opengrep`/`semgrep`, `kics scan`) and give Bob the findings as input for fixing. This is a steering-side decision; the docs give no Shell equivalent.
2. For an AI review without the package, brief Bob in Ask or Plan mode with a narrow scope (one directory, one vulnerability class, a list of files) and demand file:line evidence per finding; then verify yourself (`bob-verify-output`, `bob-modes-and-orchestration`).
3. Dependency fixes: require the worker to run the project's own test command and to report its exit code; the package's "tests at each stage" guarantee does not exist in Shell.
4. Keep secrets out of the prompt and out of the scope (`.bobignore`); general Shell rules are in `bob-security-and-data` and `bob-approvals-and-trust`.
5. If the user explicitly needs the package features (Findings panel, SafeWrite, dashboard), say it needs the Bob IDE with the license.

## Pitfalls
- Reading "Bob can scan your repo" in marketing copy as a Shell capability: the scanners run only through the IDE extension; a Shell worker only reasons over code it reads.
- Trusting an AI-only "scan" as complete: the docs themselves separate rule-based scanning from AI reasoning and note the file sweep is the exhaustive one; a Shell prompt is neither. Report coverage honestly.
- Running a large AI scan prompt on a big repo through `bob run`: cost scales with project size (Bobcoins); set `--max-cost` / `--max-turns` (`bob-cost-and-limits`).
- Code leaves the machine in both cases: package scanners send code to the Bob backend; Shell sends it to Bob inference. Check data residency (`bob-security-and-data`) before pointing either at sensitive code.
- Enterprise `EnforcedHooks` and auto-approve locks apply to the IDE; a Shell worker's approvals are governed by its own settings (`bob-approvals-and-trust`).

## See also
`bob-for-i-package`, `bob-security-and-data`, `bob-approvals-and-trust`, `bob-hooks`, `bob-cost-and-limits`, `bob-verify-output`, `bob-when-to-use`.

## Sources
`ide/premium-packages/pkg-index`, `.../devsecops/devsecops-index`, `.../getting-started`, `.../prerequisites`, `.../workflows`, `.../troubleshooting` (FAQ answers not fetched), `ide/premium-packages/updating-premium-packages`, `ide/security/bob-security-guidance`, `ide/security/data-residency`, `ide/security/group-policies`, `shell/configuration/telemetry-data-shell`; `local-help` (bob 2.0.6 `bob --help`).
