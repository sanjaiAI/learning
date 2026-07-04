# Experience Summary

> **Total Experience:** ~14 years in DevSecOps / Cloud / Platform Engineering

---

## Technical Architect (DevSecOps), Elektrobit (Oct 2021 – Present)

---

## CI/CD Architecture & Build Engineering

- Architect and maintain end-to-end CI/CD and CT pipelines for AOSP, QNX, and AUTOSAR stacks across Linux and Windows
- Design multi-stage validation workflows covering firmware image generation, signing, packaging, and release promotion
- Develop reusable Jenkins Shared Libraries (Groovy/Python) to enforce standardized, secure build and release patterns across SoC programs
- Own embedded build environment lifecycle — orchestration, debugging, dependency management, and release validation

## Security Architecture & DevSecOps

- Define and enforce DevSecOps quality gates: SAST (SonarQube), FOSS/license compliance (BlackDuck), container scanning (Trivy), and secret detection (Gitleaks)
- Architect secure artifact promotion pipelines with OPA Gatekeeper policy enforcement and secure image signing
- Design secrets management strategy using HashiCorp Vault for dynamic credential injection, eliminating static secrets
- Enforce Kubernetes admission control policies (OPA/Gatekeeper) and IaC compliance validation (Checkov) as mandatory pre-deployment gates
- Implement RBAC, workload identity, and network segmentation across all cluster deployments

## Cloud Infrastructure & Platform Engineering

- Architect and provision multi-node AKS and GKE clusters using Terraform with security-hardened configurations (workload identity, RBAC, network policies, secrets management)
- Design on-premises Kubernetes platform for containerized CT workloads with namespace isolation and resource governance
- Drive GitOps adoption (Argo CD / Flux CD) for declarative, auditable infrastructure and application delivery

## Embedded Hardware & Continuous Testing

- Architect hardware-in-the-loop CT infrastructure for embedded SoCs using Arduino, MiniProg, UART, and Radmoon peripherals
- Design automated power sequencing, flashing orchestration, and physical device test farm operations
- Define per-patch and nightly regression strategies: CTS/VTS/STS execution, release validation, and Polarion ALM traceability

## Observability & Reliability

- Design observability stack (Prometheus, Grafana, ELK, AlertManager, New Relic APM) for CI/CT metrics, centralized alerting, and release stability tracking
- Define SLIs/SLOs for pipeline reliability and build infrastructure health

## Platform & SCM Governance

- Architect and manage Jenkins, Gerrit, OpenGrok, and monitoring platforms on AKS with HA and centralized authentication
- Lead Bitbucket → Gerrit migration on Azure Cloud — repository governance, reviewer workflows, and role-based access
- Design OTA microservices architecture for connected vehicle infotainment (event-driven pipelines, cloud-to-device firmware delivery)

## AI-Driven Pipeline Intelligence

- Architect AI-assisted self-healing framework for CI/CD and CT pipelines
- Integrate RAG pipelines, MCP-based tool orchestration, vector embeddings, and LLM-driven root cause analysis
- Design contextual recovery action generation to reduce MTTR and manual troubleshooting effort

## Technical Leadership

- Lead internal enablement programs on Git, Gerrit, Jenkins, and DevSecOps best practices
- Define and govern branching strategies, credential management policies, and access control standards across engineering teams
- Drive architectural decisions for build infrastructure, security tooling, and platform modernization

---

## Technical Architect (DevSecOps), SAP HANA / Public Cloud (Jul 2017 – Oct 2021)

### Cloud-Native CI/CD & Platform Engineering
- Architect CI/CD pipelines for SAP HANA cloud services on public cloud infrastructure
- Design and implement container orchestration and microservices deployment workflows
- Build and maintain Kubernetes-based platform infrastructure for SAP workloads
- Automate infrastructure provisioning using Terraform and configuration management tools

### DevSecOps & Compliance
- Integrate security scanning (SAST, SCA, container scanning) into CI/CD pipelines
- Implement secrets management, IAM governance, and access control for cloud-native applications
- Enforce compliance policies for enterprise cloud deployments (SOC 2, ISO 27001)
- Design secure software delivery pipelines with artifact signing and promotion gates

### Cloud Operations & SRE
- Manage public cloud infrastructure (Azure/GCP/AWS) for SAP HANA workloads
- Design observability and monitoring solutions for cloud-native services
- Implement SLI/SLO frameworks and incident response automation
- Drive infrastructure-as-code adoption and GitOps delivery patterns

---

## Cloud Hosting Engineer → Senior Engineer (Oct 2012 – Jul 2017)

### Infrastructure & Hosting
- Manage and maintain cloud hosting infrastructure (bare-metal, virtualization, early cloud)
- Design and implement hosting environments for enterprise customers
- Server provisioning, capacity planning, and infrastructure scaling
- Linux/Windows server administration and hardening

### Automation & Operations
- Build automation scripts for infrastructure provisioning and deployment
- Implement monitoring and alerting for hosted environments
- Manage DNS, load balancing, SSL/TLS, and network security
- Incident management, root cause analysis, and operational runbooks

### Security & Compliance
- Implement access controls, firewall rules, and network segmentation
- Manage SSL certificates, encryption at rest/in transit
- Backup, disaster recovery planning, and business continuity
- Customer onboarding, SLA management, and capacity governance
