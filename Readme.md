# Security Workflows Framework

A centralized, reusable GitHub Actions security pipeline for enterprise use. One central repository holds all the logic — every application repo just calls it with a few lines of configuration.

---

## Table of Contents

- [Repository Structure](#repository-structure)
- [Reusable Workflow Architecture](#reusable-workflow-architecture)

---

## Repository Structure

The framework follows a clean hub-and-spoke model:

| Repository | Role |
|---|---|
| `my-org/security-workflows/` | Central hub — versioned, tag-released (`v1`, `v1.2.0`, …) |
| `my-org/<any-app-repo>/` | Spoke — a lightweight caller that delegates to the hub |

### Central Repository Layout

```
my-org/security-workflows/
└── .github/workflows/
    ├── reusable-security-suite.yml   # Orchestrator — the public entry point
    ├── reusable-sast.yml             # Semgrep + CodeQL + Slither
    ├── reusable-sca.yml              # npm/pnpm/yarn audit, pip-audit, cargo-audit,
    │                                 #   govulncheck, dependency-review
    ├── reusable-sbom.yml             # Syft (CycloneDX + SPDX)
    ├── reusable-container-scan.yml   # Trivy (+ optional Grype)
    └── reusable-iac-scan.yml         # Checkov + Trivy config
```

### Application Repository Layout

```
my-org/<any-app-repo>/
└── .github/
    ├── workflows/security-pr.yml     # Lightweight caller workflow
    └── dependabot.yml                # Keeps action pins up to date
```

### Runtime Artifact Layout

Every pipeline run produces a structured report bundle:

```
security-reports/
├── sast/
├── sca/
├── sbom/
├── containers/
├── iac/
└── summary/
```

---

## Reusable Workflow Architecture

The pipeline runs in three layers, with scanning jobs executing in parallel at the deepest layer.

```
PR to main/master
│
▼
security-pr.yml                            [Depth 1 — App repo caller]
│  uses: …/reusable-security-suite.yml@v1
▼
reusable-security-suite.yml               [Depth 2 — Orchestrator]
│
├── detect         →  Auto-detects languages and assets (no manual config needed)
│
├── sast       ──┐
├── sca          ├──  uses: ./reusable-*.yml   [Depth 3 — run in parallel]
├── sbom         │
├── container  ──┘    (only runs if a Dockerfile or image is present)
├── iac               (only runs if IaC assets are present)
│
└── summary    →  GitHub Step Summary + artifact bundle + Quality Gate
```

### What each job does

| Job | Tools | Runs when |
|---|---|---|
| **detect** | Built-in detection logic | Always |
| **sast** | Semgrep, CodeQL, Slither | Always |
| **sca** | npm/pnpm/yarn audit, pip-audit, cargo-audit, govulncheck, dependency-review | Always |
| **sbom** | Syft (CycloneDX + SPDX formats) | Always |
| **container** | Trivy, optional Grype | Dockerfile or image detected |
| **iac** | Checkov, Trivy config | IaC files detected |
| **summary** | — | Always (final step) |

> **Note:** No consumer configuration is needed for detection — the orchestrator figures out what to scan automatically.
