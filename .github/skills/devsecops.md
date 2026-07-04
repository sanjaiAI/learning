# DevSecOps, Security & IAM

## SAST (Static Application Security Testing)

| Tool | Level | Context | Status |
|------|-------|---------|--------|
| SonarQube | Advanced | Org (EB) + Industry standard | ⬜ Pending |
| Astrée | Advanced | Org (EB) — embedded C/C++ static analysis | ⬜ Pending |
| Checkmarx | Intermediate | Industry — enterprise SAST leader | ⬜ Pending |
| Semgrep | Intermediate | Industry — lightweight, rule-based | ⬜ Pending |
| Coverity (Synopsys) | Intermediate | Industry — C/C++/Java deep analysis | ⬜ Pending |
| Fortify (Micro Focus) | Awareness | Industry — enterprise SAST | ⬜ Pending |
| CodeQL (GitHub) | Intermediate | Industry — GitHub-native SAST | ⬜ Pending |
| Snyk Code | Intermediate | Industry — developer-first SAST | ⬜ Pending |

### Key Concepts
- Static code analysis for vulnerabilities without executing code
- Integration into CI pipelines as pre-merge quality gates
- Custom rule creation and suppression management
- Language-specific analyzers and taint analysis
- False positive triage and developer feedback loops
- Compliance mapping (OWASP Top 10, CWE, SANS 25)
- Abstract interpretation (Astrée) for safety-critical code (ISO 26262, DO-178C)
- Incremental analysis and PR-level differential scanning

---

## DAST (Dynamic Application Security Testing)

| Tool | Level | Context | Status |
|------|-------|---------|--------|
| OWASP ZAP | Advanced | Industry — open-source standard | ⬜ Pending |
| Burp Suite (PortSwigger) | Intermediate | Industry — pen testing & DAST | ⬜ Pending |
| Nikto | Intermediate | Industry — web server scanner | ⬜ Pending |
| Nuclei (ProjectDiscovery) | Intermediate | Industry — template-based scanner | ⬜ Pending |
| HCL AppScan | Awareness | Industry — enterprise DAST | ⬜ Pending |
| Qualys WAS | Awareness | Industry — cloud DAST | ⬜ Pending |

### Key Concepts
- Runtime vulnerability scanning against running applications
- Authenticated and unauthenticated scan configurations
- API security testing (REST, GraphQL, gRPC)
- Integration into CI/CD as post-deployment validation gates
- Crawling strategies and scan policies
- Finding correlation with SAST results for prioritization
- IAST (Interactive AST) for runtime instrumentation (Contrast Security)
- Fuzzing for protocol and API edge-case discovery

---

## SCA (Software Composition Analysis), Container & IaC Security

| Tool | Level | Context | Status |
|------|-------|---------|--------|
| BlackDuck (Synopsys) | Advanced | Org (EB) + Industry — FOSS/license | ⬜ Pending |
| Snyk Open Source | Intermediate | Industry — SCA leader | ⬜ Pending |
| Dependabot / Renovate | Advanced | Industry — automated dependency updates | ⬜ Pending |
| Trivy | Advanced | Industry — container/IaC/SBOM scanner | ⬜ Pending |
| Grype + Syft (Anchore) | Intermediate | Industry — SBOM generation + vuln matching | ⬜ Pending |
| Gitleaks | Advanced | Industry — secret detection | ⬜ Pending |
| TruffleHog | Intermediate | Industry — secret detection | ⬜ Pending |
| Checkov (Prisma Cloud) | Advanced | Industry — IaC policy-as-code | ⬜ Pending |
| tfsec / Terrascan | Intermediate | Industry — Terraform-specific scanning | ⬜ Pending |
| OPA Gatekeeper | Expert | Industry — Kubernetes admission control | ⬜ Pending |
| Kyverno | Intermediate | Industry — K8s-native policy engine | ⬜ Pending |
| Cosign / Sigstore | Intermediate | Industry — image signing & verification | ⬜ Pending |

---

## Secrets & IAM

| Tool | Level | Status |
|------|-------|--------|
| HashiCorp Vault | Advanced | ⬜ Pending |
| Azure Key Vault | Advanced | ⬜ Pending |
| GCP Secret Manager / Cloud KMS | Advanced | ⬜ Pending |
| Keycloak (OIDC/SAML) | Intermediate | ⬜ Pending |
| Azure AD / LDAP / SSO | Advanced | ⬜ Pending |

---

## Key Competencies

- DevSecOps quality gate architecture (shift-left security)
- SAST pipeline integration: pre-merge blocking, incremental analysis, baseline management
- DAST pipeline integration: post-deploy scanning, authenticated crawls, API fuzzing
- IAST for runtime coverage in staging/QA environments
- SCA for open-source risk: license compliance, CVE tracking, SBOM (SPDX/CycloneDX)
- Container scanning pipeline integration (image layers, base image policies)
- Supply chain security (SLSA framework, Sigstore, in-toto attestations)
- OPA Gatekeeper / Kyverno constraint templates and admission control policies
- Secrets management strategy (Vault dynamic secrets, KMS encryption, rotation)
- Secure artifact promotion with image signing and policy enforcement
- IAM architecture (OIDC/SAML federation, RBAC, workload identity, zero-trust)
- IaC compliance validation as pre-deployment gates
- Secret detection and credential rotation automation
- Security metrics: vulnerability SLAs, MTTR, escape rate, risk scoring
- Compliance frameworks: SOC 2, ISO 27001, PCI-DSS, GDPR, ISO 26262 (automotive)
- Vulnerability management lifecycle: triage → prioritize → remediate → verify
