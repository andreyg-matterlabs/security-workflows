1. Repository structure

The framework uses the standard enterprise split: one central repository holds
the reusable workflows; every application repository ships a tiny caller.

textmy-org/security-workflows/                 # central, versioned, tag-released (v1, v1.2.0, …)
└── .github/workflows/
    ├── reusable-security-suite.yml        # orchestrator (the public entrypoint)
    ├── reusable-sast.yml                   # Semgrep + CodeQL + Slither
    ├── reusable-sca.yml                    # npm/pnpm/yarn audit, pip-audit, cargo-audit, govulncheck, dependency-review
    ├── reusable-sbom.yml                   # Syft (CycloneDX + SPDX)
    ├── reusable-container-scan.yml         # Trivy (+ optional Grype)
    └── reusable-iac-scan.yml               # Checkov + Trivy config

my-org/<any-app-repo>/                      # every consuming repo
└── .github/
    ├── workflows/security-pr.yml           # lightweight caller
    └── dependabot.yml                      # keeps action pins fresh

Runtime artifact layout produced for every run:

textsecurity-reports/
├── sast/         ├── containers/
├── sca/          ├── iac/
├── sbom/         └── summary/


2. Reusable workflow architecture

text PR to main/master
   │
   ▼
security-pr.yml  (caller, in app repo)         depth 1
   │  uses: …/reusable-security-suite.yml@v1
   ▼
reusable-security-suite.yml  (orchestrator)    depth 2
   │
   ├── detect            → languages/assets present (no consumer config needed)
   ├── sast      ─┐
   ├── sca         ├─ uses: ./reusable-*.yml   depth 3   (run in parallel)
   ├── sbom        │
   ├── container  ─┘  (conditional on Dockerfile/image)
   ├── iac           (conditional on IaC assets)
   │
   └── summary    → Step Summary + all-security-reports bundle + QUALITY GATE

