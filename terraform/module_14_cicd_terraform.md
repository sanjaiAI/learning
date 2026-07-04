# Module 14: CI/CD for Terraform

## 📖 Story: The Assembly Line Quality Control

**English:**
Think of Terraform CI/CD like a car manufacturing assembly line. You wouldn't let every worker on the floor just weld things together and hope the car works. Instead:

1. **Design review** (PR review) — someone checks the blueprint
2. **Dry run** (terraform plan) — simulate assembly without using real parts
3. **Quality gate** (approval) — supervisor signs off
4. **Production** (terraform apply) — actually build the car

Without CI/CD for Terraform, you have 50 developers doing `terraform apply` from their laptops. That means:
- No audit trail (who changed what?)
- No review (what if the change breaks production?)
- No consistency (different Terraform versions, different provider versions)
- Credential sprawl (everyone has admin credentials locally)

At EB, we built a Jenkins pipeline that enforced: **plan on PR → human approval → apply on merge.** No developer ever ran `terraform apply` locally for production. Period.

The senior architect question isn't "can you write a pipeline?" — it's "can you design a pipeline that works for 50 developers across 5 environments with proper governance?"

**தமிழ்:**
Terraform CI/CD-ஐ car manufacturing assembly line போல நினைக்கவும். ஒவ்வொரு worker-ம் தானாக weld செய்து car work ஆகும் என்று hope செய்ய மாட்டீர்கள்:

1. **Design review** (PR review) — blueprint-ஐ யாரோ check செய்கிறார்
2. **Dry run** (terraform plan) — real parts use செய்யாமல் simulate
3. **Quality gate** (approval) — supervisor sign off
4. **Production** (terraform apply) — actually build

CI/CD இல்லாமல், 50 developers laptops-லிருந்து `terraform apply` செய்கிறார்கள். அதாவது:
- Audit trail இல்லை (யார் என்ன change?)
- Review இல்லை (production break ஆனால்?)
- Consistency இல்லை (different versions)
- Credential sprawl (எல்லோரிடமும் admin credentials)

EB-ல், Jenkins pipeline build செய்தோம்: **plan on PR → human approval → apply on merge.** Production-க்கு எந்த developer-ம் locally `terraform apply` run செய்யவில்லை.

Senior architect கேள்வி: "pipeline எழுத முடியுமா?" அல்ல — "50 developers, 5 environments, proper governance-உடன் pipeline design செய்ய முடியுமா?"

---

## 📊 Architecture & Concepts

### The Standard Terraform CI/CD Pattern

```
┌─────────────────────────────────────────────────────────────────────┐
│              TERRAFORM CI/CD — THE GOLDEN PATTERN                     │
│                                                                       │
│  Developer                    CI System                 Cloud         │
│     │                            │                        │          │
│     │ 1. Push branch             │                        │          │
│     │───────────────────────────▶│                        │          │
│     │                            │ 2. terraform init      │          │
│     │                            │ 3. terraform validate  │          │
│     │                            │ 4. security scan       │          │
│     │                            │ 5. terraform plan      │          │
│     │                            │────────────────────────▶          │
│     │                            │◀────────────────────────          │
│     │ 6. Plan output as          │    (plan output)       │          │
│     │    PR comment              │                        │          │
│     │◀───────────────────────────│                        │          │
│     │                            │                        │          │
│     │ 7. Review + Approve        │                        │          │
│     │───────────────────────────▶│                        │          │
│     │                            │                        │          │
│     │ 8. Merge to main           │                        │          │
│     │───────────────────────────▶│ 9. terraform apply     │          │
│     │                            │────────────────────────▶          │
│     │                            │◀────────────────────────          │
│     │                            │   (apply complete)     │          │
│     │ 10. Notification           │                        │          │
│     │◀───────────────────────────│                        │          │
└─────────────────────────────────────────────────────────────────────┘
```

### Authentication: OIDC vs Long-lived Credentials

```
┌────────────────────────────────────────────────────────────────┐
│           AUTHENTICATION APPROACHES                              │
│                                                                  │
│  ❌ BAD: Long-lived credentials                                 │
│  ┌────────────────────────────────────────────────┐            │
│  │ CI System stores GCP_SERVICE_ACCOUNT_KEY        │            │
│  │ → Never expires                                 │            │
│  │ → If leaked, attacker has permanent access      │            │
│  │ → Hard to rotate across all pipelines           │            │
│  └────────────────────────────────────────────────┘            │
│                                                                  │
│  ✅ GOOD: OIDC (Workload Identity Federation)                   │
│  ┌────────────────────────────────────────────────┐            │
│  │ CI System → "I am GitHub Actions run #1234"     │            │
│  │ GCP → "I trust GitHub Actions for this repo"    │            │
│  │ GCP → "Here's a 1-hour token"                   │            │
│  │ → No stored secrets                             │            │
│  │ → Auto-expires                                  │            │
│  │ → Scoped to specific repo/branch                │            │
│  └────────────────────────────────────────────────┘            │
│                                                                  │
│  OIDC Flow:                                                     │
│  CI Run → Request OIDC token → Present to GCP WIF →            │
│  GCP validates issuer → Issues short-lived credential →         │
│  Terraform uses credential → Expires after run                  │
└────────────────────────────────────────────────────────────────┘
```

### CI/CD Tools Comparison

| Feature | GitHub Actions | Jenkins | Atlantis | GitLab CI |
|---------|---------------|---------|----------|-----------|
| PR plan comments | Native | Plugin needed | Built-in | Native |
| OIDC support | Native | Plugin | N/A (runs plans directly) | Native |
| Approval gates | Environment protection | Build approval | PR approval = apply | Environment protection |
| Parallel envs | Matrix strategy | Parallel stages | Workspace-based | Parallel jobs |
| State locking | External | External | Built-in | External |
| Cost | Free (public) / Paid | Self-hosted (free) | Self-hosted (free) | Free tier / Paid |
| Best for | GitHub-native teams | Enterprise/complex | Terraform-focused | GitLab-native teams |

### Drift Detection Architecture

```
┌────────────────────────────────────────────────────────────────┐
│              DRIFT DETECTION PATTERN                             │
│                                                                  │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐   │
│  │  Scheduled   │────▶│  terraform   │────▶│   Compare    │   │
│  │  Cron Job    │     │    plan      │     │   Output     │   │
│  │ (every 4hr)  │     │              │     │              │   │
│  └──────────────┘     └──────────────┘     └──────┬───────┘   │
│                                                     │           │
│                              ┌───────────────────────┤           │
│                              │                       │           │
│                         No changes              Changes found   │
│                              │                       │           │
│                              ▼                       ▼           │
│                         ┌────────┐           ┌────────────┐    │
│                         │  Done  │           │   Alert!   │    │
│                         └────────┘           │ Slack/PD   │    │
│                                              └────────────┘    │
│                                                     │           │
│                              ┌───────────────────────┤           │
│                              │                       │           │
│                         Auto-fix              Manual review     │
│                      (terraform apply)      (investigate why)   │
└────────────────────────────────────────────────────────────────┘
```

---

## 🧠 Byheart for Interview

### The CI/CD Terraform Mantra

```
"Plan on PR, Apply on Merge, Detect Drift Daily"

Pipeline Stages (memorize this order):
1. init        → Download providers, configure backend
2. validate    → Syntax check
3. scan        → Security (Checkov/tfsec)
4. plan        → Show what will change
5. [comment]   → Post plan to PR
6. [approve]   → Human gate (prod only)
7. apply       → Make the changes
8. [notify]    → Slack/Teams notification

Key Principle: "No human should ever run terraform apply locally for production"
```

### OIDC Key Points

```
OIDC = "Prove who you are without sharing secrets"

GitHub Actions → GCP:
  - GitHub issues OIDC token (JWT) per workflow run
  - Token contains: repo, branch, workflow, run_id
  - GCP Workload Identity Federation validates token
  - GCP issues short-lived access token (1 hour)
  - NO service account key stored anywhere

Why OIDC matters (interview gold):
  - No credential rotation needed
  - No secret leakage risk
  - Scoped: only specific repo+branch can authenticate
  - Auditable: every token issuance is logged
```

### Pipeline Design for Scale (50+ developers)

```
Architecture Decision Record:
├── State: Remote backend (GCS) with locking
├── Auth: OIDC (no long-lived secrets)
├── Branch strategy: feature → main (plan on PR, apply on merge)
├── Environments: Directory-based (envs/dev, envs/staging, envs/prod)
├── Approval: Auto-apply dev/staging, manual approval for prod
├── Notifications: Slack channel per environment
├── Drift: Scheduled plan every 4 hours
└── Rollback: Revert PR → triggers apply of previous state
```

---

## ⚡ Quick Hands-on

### Lab Setup

```bash
# SSH to lab server
ssh root@203.57.85.108

# Create lab directory
mkdir -p ~/tf-lab/cicd && cd ~/tf-lab/cicd
```

### Exercise 1: GitHub Actions Pipeline

```bash
# Create a complete GitHub Actions workflow
mkdir -p .github/workflows

cat > .github/workflows/terraform.yml << 'EOF'
name: Terraform CI/CD

on:
  pull_request:
    branches: [main]
    paths: ['terraform/**']
  push:
    branches: [main]
    paths: ['terraform/**']

permissions:
  contents: read
  pull-requests: write       # For PR comments
  id-token: write            # For OIDC

env:
  TF_VERSION: "1.7.0"
  WORKING_DIR: "terraform/environments/dev"

jobs:
  plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      # OIDC Authentication — no stored secrets!
      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: 'projects/123456/locations/global/workloadIdentityPools/github-pool/providers/github-provider'
          service_account: 'terraform-ci@my-project.iam.gserviceaccount.com'

      - name: Terraform Init
        working-directory: ${{ env.WORKING_DIR }}
        run: terraform init -input=false

      - name: Terraform Validate
        working-directory: ${{ env.WORKING_DIR }}
        run: terraform validate

      - name: Security Scan
        working-directory: ${{ env.WORKING_DIR }}
        run: |
          pip install checkov
          checkov -d . --quiet --compact

      - name: Terraform Plan
        id: plan
        working-directory: ${{ env.WORKING_DIR }}
        run: |
          terraform plan -no-color -input=false -out=tfplan 2>&1 | tee plan_output.txt
        continue-on-error: true

      # Post plan output as PR comment
      - name: Comment PR with Plan
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('${{ env.WORKING_DIR }}/plan_output.txt', 'utf8');
            const truncated = plan.length > 60000 ? plan.substring(0, 60000) + '\n... (truncated)' : plan;
            
            const body = `## Terraform Plan Output
            
            \`\`\`
            ${truncated}
            \`\`\`
            
            **Status:** ${{ steps.plan.outcome }}
            `;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: body
            });

      - name: Fail if plan failed
        if: steps.plan.outcome == 'failure'
        run: exit 1

  apply:
    name: Terraform Apply
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment: production  # Requires approval!
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: 'projects/123456/locations/global/workloadIdentityPools/github-pool/providers/github-provider'
          service_account: 'terraform-ci@my-project.iam.gserviceaccount.com'

      - name: Terraform Init
        working-directory: ${{ env.WORKING_DIR }}
        run: terraform init -input=false

      - name: Terraform Apply
        working-directory: ${{ env.WORKING_DIR }}
        run: terraform apply -auto-approve -input=false
EOF

echo "GitHub Actions workflow created!"
```

### Exercise 2: Jenkins Pipeline (EB Pattern)

```bash
cat > Jenkinsfile << 'EOF'
// Jenkinsfile — Terraform CI/CD Pipeline (EB Project Pattern)
pipeline {
    agent { label 'terraform' }
    
    environment {
        TF_VERSION     = '1.7.0'
        TF_DIR         = 'terraform/environments'
        GCP_PROJECT    = credentials('gcp-project-id')
        // OIDC: Jenkins uses Workload Identity, no key file
    }
    
    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'staging', 'prod'],
            description: 'Target environment'
        )
        booleanParam(
            name: 'AUTO_APPLY',
            defaultValue: false,
            description: 'Auto-apply without manual approval (dev/staging only)'
        )
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Init') {
            steps {
                dir("${TF_DIR}/${params.ENVIRONMENT}") {
                    sh '''
                        terraform init \
                            -input=false \
                            -backend-config="bucket=${GCP_PROJECT}-tf-state" \
                            -backend-config="prefix=${ENVIRONMENT}"
                    '''
                }
            }
        }
        
        stage('Validate') {
            steps {
                dir("${TF_DIR}/${params.ENVIRONMENT}") {
                    sh 'terraform validate'
                }
            }
        }
        
        stage('Security Scan') {
            parallel {
                stage('Checkov') {
                    steps {
                        dir("${TF_DIR}/${params.ENVIRONMENT}") {
                            sh '''
                                checkov -d . \
                                    --config-file .checkov.yaml \
                                    --output junitxml \
                                    --output-file-path results/
                            '''
                        }
                    }
                    post {
                        always {
                            junit "${TF_DIR}/${params.ENVIRONMENT}/results/*.xml"
                        }
                    }
                }
                stage('tfsec') {
                    steps {
                        dir("${TF_DIR}/${params.ENVIRONMENT}") {
                            sh 'tfsec . --minimum-severity HIGH --soft-fail'
                        }
                    }
                }
            }
        }
        
        stage('Plan') {
            steps {
                dir("${TF_DIR}/${params.ENVIRONMENT}") {
                    sh '''
                        terraform plan \
                            -input=false \
                            -out=tfplan \
                            -no-color | tee plan_output.txt
                    '''
                    // Archive plan for audit
                    archiveArtifacts artifacts: 'tfplan,plan_output.txt'
                }
            }
        }
        
        stage('Approval') {
            when {
                expression { 
                    params.ENVIRONMENT == 'prod' || !params.AUTO_APPLY 
                }
            }
            steps {
                script {
                    def planOutput = readFile("${TF_DIR}/${params.ENVIRONMENT}/plan_output.txt")
                    // Send to Slack for visibility
                    slackSend(
                        channel: "#terraform-${params.ENVIRONMENT}",
                        message: "Terraform plan ready for ${params.ENVIRONMENT}:\n```${planOutput.take(3000)}```\nApprove in Jenkins: ${BUILD_URL}"
                    )
                    
                    input(
                        message: "Apply Terraform to ${params.ENVIRONMENT}?",
                        submitter: 'platform-team,tech-leads'
                    )
                }
            }
        }
        
        stage('Apply') {
            steps {
                dir("${TF_DIR}/${params.ENVIRONMENT}") {
                    sh 'terraform apply -auto-approve -input=false tfplan'
                }
            }
            post {
                success {
                    slackSend(
                        channel: "#terraform-${params.ENVIRONMENT}",
                        color: 'good',
                        message: "✅ Terraform apply SUCCESS for ${params.ENVIRONMENT} (${BUILD_URL})"
                    )
                }
                failure {
                    slackSend(
                        channel: "#terraform-${params.ENVIRONMENT}",
                        color: 'danger',
                        message: "❌ Terraform apply FAILED for ${params.ENVIRONMENT} (${BUILD_URL})"
                    )
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}
EOF

echo "Jenkinsfile created!"
```

### Exercise 3: OIDC Setup (GCP Workload Identity Federation)

```bash
cat > oidc_setup.tf << 'EOF'
# Setup OIDC for GitHub Actions → GCP
# This eliminates the need for service account keys in CI

# Workload Identity Pool
resource "google_iam_workload_identity_pool" "github" {
  project                   = var.project_id
  workload_identity_pool_id = "github-actions-pool"
  display_name              = "GitHub Actions Pool"
  description               = "Pool for GitHub Actions OIDC"
}

# Workload Identity Provider (GitHub)
resource "google_iam_workload_identity_pool_provider" "github" {
  project                            = var.project_id
  workload_identity_pool_id          = google_iam_workload_identity_pool.github.workload_identity_pool_id
  workload_identity_pool_provider_id = "github-provider"
  display_name                       = "GitHub Actions Provider"

  # Map GitHub token attributes to Google attributes
  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.actor"      = "assertion.actor"
    "attribute.repository" = "assertion.repository"
    "attribute.ref"        = "assertion.ref"
  }

  # Only trust tokens from GitHub
  attribute_condition = "assertion.repository_owner == '${var.github_org}'"

  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }
}

# Service account that Terraform CI will impersonate
resource "google_service_account" "terraform_ci" {
  project      = var.project_id
  account_id   = "terraform-ci"
  display_name = "Terraform CI Service Account"
  description  = "Used by GitHub Actions via OIDC for Terraform operations"
}

# Allow GitHub Actions to impersonate this SA
# SCOPED: Only specific repo can use this
resource "google_service_account_iam_member" "github_actions" {
  service_account_id = google_service_account.terraform_ci.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.repository/${var.github_org}/${var.github_repo}"
}

# Grant the SA the permissions it needs
resource "google_project_iam_member" "terraform_ci_roles" {
  for_each = toset([
    "roles/compute.admin",
    "roles/storage.admin",
    "roles/container.admin",
    "roles/iam.securityAdmin",
  ])

  project = var.project_id
  role    = each.value
  member  = "serviceAccount:${google_service_account.terraform_ci.email}"
}

variable "project_id" {
  type = string
}

variable "github_org" {
  type    = string
  default = "my-org"
}

variable "github_repo" {
  type    = string
  default = "infra-terraform"
}

output "workload_identity_provider" {
  value = google_iam_workload_identity_pool_provider.github.name
  description = "Use this value in GitHub Actions auth step"
}
EOF

echo "OIDC Terraform configuration created!"
```

### Exercise 4: Drift Detection Cron

```bash
cat > .github/workflows/drift-detection.yml << 'EOF'
name: Drift Detection

on:
  schedule:
    - cron: '0 */4 * * *'  # Every 4 hours
  workflow_dispatch:         # Manual trigger

permissions:
  contents: read
  id-token: write
  issues: write

env:
  TF_VERSION: "1.7.0"

jobs:
  detect-drift:
    name: Check for Drift
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging, prod]
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: 'projects/123456/locations/global/workloadIdentityPools/github-pool/providers/github-provider'
          service_account: 'terraform-ci@my-project.iam.gserviceaccount.com'

      - name: Terraform Init
        working-directory: terraform/environments/${{ matrix.environment }}
        run: terraform init -input=false

      - name: Detect Drift
        id: drift
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          terraform plan -detailed-exitcode -input=false -no-color > drift_output.txt 2>&1 || EXIT_CODE=$?
          
          if [ "${EXIT_CODE}" = "2" ]; then
            echo "drift_detected=true" >> $GITHUB_OUTPUT
            echo "::warning::Drift detected in ${{ matrix.environment }}!"
          else
            echo "drift_detected=false" >> $GITHUB_OUTPUT
          fi

      - name: Create Issue on Drift
        if: steps.drift.outputs.drift_detected == 'true'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const drift = fs.readFileSync(
              'terraform/environments/${{ matrix.environment }}/drift_output.txt', 'utf8'
            );
            
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `🚨 Drift detected in ${{ matrix.environment }}`,
              body: `## Drift Detection Alert\n\nEnvironment: **${{ matrix.environment }}**\nDetected: ${new Date().toISOString()}\n\n\`\`\`\n${drift.substring(0, 60000)}\n\`\`\`\n\nAction needed: Investigate manual changes and reconcile.`,
              labels: ['drift', 'infrastructure', '${{ matrix.environment }}']
            });

      - name: Notify Slack on Drift
        if: steps.drift.outputs.drift_detected == 'true'
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "🚨 Infrastructure drift detected in *${{ matrix.environment }}*! Check GitHub Issues."
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
EOF

echo "Drift detection workflow created!"
```

### Exercise 5: Atlantis Configuration

```bash
cat > atlantis.yaml << 'EOF'
# atlantis.yaml — repo-level config
version: 3
projects:
  - name: dev
    dir: terraform/environments/dev
    workspace: default
    terraform_version: v1.7.0
    autoplan:
      when_modified: ["*.tf", "*.tfvars", "../modules/**"]
      enabled: true
    apply_requirements: [approved]
    
  - name: staging
    dir: terraform/environments/staging
    workspace: default
    terraform_version: v1.7.0
    autoplan:
      when_modified: ["*.tf", "*.tfvars", "../modules/**"]
      enabled: true
    apply_requirements: [approved, mergeable]
    
  - name: prod
    dir: terraform/environments/prod
    workspace: default
    terraform_version: v1.7.0
    autoplan:
      when_modified: ["*.tf", "*.tfvars", "../modules/**"]
      enabled: true
    apply_requirements: [approved, mergeable]
    workflow: prod-workflow

workflows:
  prod-workflow:
    plan:
      steps:
        - init
        - run: checkov -d . --quiet
        - plan:
            extra_args: ["-var-file=prod.tfvars"]
    apply:
      steps:
        - run: echo "Applying to PRODUCTION — verified by ${PULL_AUTHOR}"
        - apply
EOF

echo "Atlantis configuration created!"
echo ""
echo "Atlantis usage:"
echo "  PR comment: 'atlantis plan -p dev'     → runs plan for dev"
echo "  PR comment: 'atlantis apply -p dev'    → applies dev (after approval)"
echo "  PR comment: 'atlantis plan'            → plans all modified projects"
```

---

## 🔥 Scenario Challenge

### Scenario: "Design a Terraform CI/CD Pipeline for 50 Developers, 5 Environments"

**The Interview Setup:**
"You're joining a company with 50 developers who currently run Terraform locally. They have dev, staging, prod, DR, and sandbox environments. Design the CI/CD pipeline."

**Senior Architect Answer:**

```
┌────────────────────────────────────────────────────────────────────┐
│              PIPELINE DESIGN FOR 50 DEVELOPERS                      │
│                                                                      │
│  PRINCIPLES:                                                        │
│  1. Developers never run apply locally                              │
│  2. All changes via PR (audit trail)                                │
│  3. Environments are isolated (separate state, separate auth)       │
│  4. Prod requires human approval                                    │
│  5. Drift detected automatically                                    │
│                                                                      │
│  ARCHITECTURE:                                                      │
│                                                                      │
│  ┌──────────┐    ┌───────────────────────────────────────┐         │
│  │ Developer │───▶│ Git (feature branch)                   │         │
│  └──────────┘    └───────────┬───────────────────────────┘         │
│                              │ PR Created                           │
│                              ▼                                      │
│                 ┌─────────────────────────────┐                     │
│                 │ CI Pipeline (per environment)│                     │
│                 │ ├── init                     │                     │
│                 │ ├── validate                 │                     │
│                 │ ├── checkov scan             │                     │
│                 │ ├── plan                     │                     │
│                 │ └── PR comment with plan     │                     │
│                 └─────────────┬───────────────┘                     │
│                              │ Merge to main                        │
│                              ▼                                      │
│                 ┌─────────────────────────────┐                     │
│                 │ CD Pipeline                  │                     │
│                 │ ├── dev: auto-apply          │                     │
│                 │ ├── staging: auto-apply      │                     │
│                 │ ├── prod: MANUAL APPROVAL    │                     │
│                 │ └── DR: follows prod         │                     │
│                 └─────────────────────────────┘                     │
│                                                                      │
│  GUARDRAILS:                                                        │
│  ├── OIDC auth (no stored credentials)                             │
│  ├── Checkov gate (no merge if HIGH violations)                    │
│  ├── Plan size limit (>50 resources = extra review)                │
│  ├── Blast radius check (no destroy >5 resources without approval) │
│  └── State locking (prevent concurrent applies)                    │
│                                                                      │
│  OBSERVABILITY:                                                     │
│  ├── Drift detection every 4 hours                                 │
│  ├── Slack notifications per environment                           │
│  ├── Cost estimation in PR (Infracost)                             │
│  └── Audit log: who applied what, when                             │
└────────────────────────────────────────────────────────────────────┘
```

**Key Design Decisions (explain in interview):**

```
Q: "Why directory-based instead of workspaces?"
A: "Isolation. Each environment has its own state file, own CI job,
    own approval process. Workspaces share code — one mistake affects all."

Q: "Why OIDC instead of service account keys?"
A: "Zero credential management. No rotation, no leakage risk, 
    scoped to specific repo+branch. It's the modern standard."

Q: "Why not just Atlantis?"
A: "Atlantis is great for pure Terraform, but we needed integration
    with existing Jenkins (build + test + deploy all in one pipeline).
    Atlantis also lacks the security scanning integration we needed."

Q: "How do you prevent one developer from breaking prod?"
A: "Multiple layers: PR review, security scan gate, plan review,
    manual approval for prod, blast radius checks. Defense in depth."
```

---

## 🏗️ Real Project Reference: EB Jenkins Pipeline

### EB Project — Actual Pipeline Design

```
┌────────────────────────────────────────────────────────────────┐
│                 EB PROJECT — TERRAFORM PIPELINE                  │
│                                                                  │
│  Context: Embedded CI/CT system on GCP                          │
│  Team: 50+ developers                                           │
│  Environments: dev, staging, prod (GKE + Cloud SQL + GCS)       │
│  Tool: Jenkins (existing CI, mandated by org)                   │
│                                                                  │
│  Pipeline Flow:                                                 │
│  ┌──────────┐                                                   │
│  │ Trigger  │──── PR created/updated → plan only               │
│  │          │──── Merge to main → plan + apply (with gates)    │
│  └──────────┘                                                   │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────────────────────────────────┐              │
│  │ Stage: Init + Validate                        │              │
│  │  terraform init -backend-config=env.hcl      │              │
│  │  terraform validate                           │              │
│  └──────────────────────────────────────────────┘              │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────────────────────────────────┐              │
│  │ Stage: Security Gate (BLOCKING)               │              │
│  │  checkov -d . --config-file .checkov.yaml    │              │
│  │  → Fail build on HIGH/CRITICAL               │              │
│  └──────────────────────────────────────────────┘              │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────────────────────────────────┐              │
│  │ Stage: Plan                                   │              │
│  │  terraform plan -out=tfplan                   │              │
│  │  → Archive plan artifact                      │              │
│  │  → Post plan to Slack                         │              │
│  └──────────────────────────────────────────────┘              │
│       │                                                         │
│       ▼ (prod only)                                            │
│  ┌──────────────────────────────────────────────┐              │
│  │ Stage: Manual Approval                        │              │
│  │  → Slack notification to #platform-approvals  │              │
│  │  → Only tech-leads can approve                │              │
│  │  → 4-hour timeout                            │              │
│  └──────────────────────────────────────────────┘              │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────────────────────────────────┐              │
│  │ Stage: Apply                                  │              │
│  │  terraform apply -auto-approve tfplan         │              │
│  │  → Post result to Slack                       │              │
│  │  → Tag commit with applied version            │              │
│  └──────────────────────────────────────────────┘              │
│                                                                  │
│  Results:                                                       │
│  ├── Zero manual applies in 6 months                           │
│  ├── Security findings: 47 → 0 in 3 weeks                     │
│  ├── Average PR to deploy: 45 minutes (dev), 2 hours (prod)   │
│  └── Zero accidental production incidents                      │
└────────────────────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A

### Q1: "Design a Terraform CI/CD pipeline for a team of 50 developers"

**English Answer:**
"I'd design around four principles: isolation, automation, governance, and observability.

**Structure:** Directory-based environments (not workspaces) — each env has its own state, CI job, and approval chain. Shared modules in a separate directory.

**Pipeline:** Plan runs on every PR with results posted as PR comments. Security scan (Checkov) is a blocking gate. Apply triggers on merge to main — auto-apply for dev/staging, manual approval for prod.

**Auth:** OIDC (Workload Identity Federation) — no long-lived credentials stored anywhere. Each environment's CI job authenticates with its own scoped identity.

**Guardrails:** Blast radius detection (flag if >10 resources changing), cost estimation in PR (Infracost), drift detection every 4 hours.

At EB, this pipeline design resulted in zero manual applies and zero accidental production changes over 6 months."

**தமிழ் Answer:**
"நான்கு principles-ஐ சுற்றி design செய்வேன்: isolation, automation, governance, observability.

**Structure:** Directory-based environments — ஒவ்வொரு env-க்கும் own state, CI job, approval chain. Shared modules separate directory-ல்.

**Pipeline:** ஒவ்வொரு PR-லும் plan run ஆகும், results PR comments-ஆக post. Security scan (Checkov) blocking gate. Main-க்கு merge ஆனால் apply trigger — dev/staging auto-apply, prod manual approval.

**Auth:** OIDC — long-lived credentials எங்கும் store ஆகாது. ஒவ்வொரு environment-ன் CI job own scoped identity-உடன் authenticate.

**Guardrails:** Blast radius detection, cost estimation, drift detection.

EB-ல், இந்த pipeline design 6 மாதங்களில் zero manual applies, zero accidental production changes."

---

### Q2: "How do you handle Terraform state locking in CI/CD?"

**English Answer:**
"State locking is critical when multiple CI runs could target the same environment simultaneously:

1. **Backend-native locking:** GCS backend uses a `.tflock` file. If another process has the lock, Terraform waits (or fails after timeout).

2. **CI-level protection:** Only one apply job per environment can run at a time. In Jenkins, use `disableConcurrentBuilds()`. In GitHub Actions, use `concurrency` groups.

3. **Force-unlock procedure:** If a CI job crashes mid-apply and leaves a stale lock, we have a documented procedure: verify no apply is running → `terraform force-unlock <LOCK_ID>` → investigate why the crash happened.

The key point: locking must happen at BOTH the Terraform state level AND the CI pipeline level."

**தமிழ் Answer:**
"Multiple CI runs ஒரே environment-ஐ simultaneously target செய்யும்போது state locking critical:

1. **Backend-native locking:** GCS backend `.tflock` file use செய்கிறது. மற்றொரு process lock வைத்திருந்தால், Terraform wait செய்கிறது.

2. **CI-level protection:** ஒரு environment-க்கு ஒரே நேரத்தில் ஒரு apply job மட்டுமே run ஆகும். Jenkins-ல் `disableConcurrentBuilds()`, GitHub Actions-ல் `concurrency` groups.

3. **Force-unlock:** CI job apply-ன் நடுவில் crash ஆனால், documented procedure: verify → `terraform force-unlock` → investigate.

Key point: locking Terraform state level-லும் CI pipeline level-லும் happen ஆக வேண்டும்."

---

### Q3: "What's your rollback strategy for Terraform?"

**English Answer:**
"Terraform doesn't have a built-in rollback button — it's declarative, not imperative. My strategy:

1. **Revert the code:** Create a revert PR that undoes the problematic change. When merged, CI runs plan/apply which returns infrastructure to previous state.

2. **State-level recovery:** If state is corrupted, restore from the versioned backend (GCS bucket versioning enabled). This is why we always enable bucket versioning on state storage.

3. **Prevent needing rollback:** The plan-on-PR pattern means we see changes BEFORE apply. Combined with blast radius checks, we rarely need emergency rollbacks.

4. **Blue-green for critical resources:** For databases and stateful resources, we don't do in-place changes — we provision new, migrate, then decommission old.

The architect mindset: rollback is the last resort, not the primary strategy. Prevention > Detection > Rollback."

**தமிழ் Answer:**
"Terraform-ல் built-in rollback button இல்லை — declarative, imperative அல்ல. என் strategy:

1. **Code revert:** Revert PR create → merge → CI plan/apply → infrastructure previous state-க்கு return.

2. **State recovery:** State corrupt ஆனால், versioned backend-லிருந்து restore (GCS bucket versioning).

3. **Rollback தேவையை prevent:** Plan-on-PR pattern — changes-ஐ apply-க்கு முன்பே பார்க்கிறோம்.

4. **Blue-green:** Databases, stateful resources-க்கு in-place changes செய்வதில்லை — new provision, migrate, old decommission.

Architect mindset: rollback last resort. Prevention > Detection > Rollback."

---

### Q4: "How do you handle secrets in Terraform CI/CD?"

**English Answer:**
"Secrets in Terraform CI/CD are handled at three levels:

1. **CI Authentication:** OIDC — no stored credentials. The CI system proves its identity and gets short-lived tokens.

2. **Terraform variables (sensitive):** Stored in the CI system's secret store (Jenkins credentials, GitHub Secrets). Injected as environment variables (`TF_VAR_db_password`). Never in code or tfvars files.

3. **State file secrets:** State contains sensitive values in plaintext — this is why state backend MUST have encryption at rest and strict access controls. Only the CI service account can read state.

4. **Output secrets:** If Terraform outputs contain secrets (e.g., generated passwords), mark them `sensitive = true` and don't log them in CI output.

The golden rule: secrets exist in exactly TWO places — the secrets manager and the running infrastructure. Never in Git, never in CI logs."

**தமிழ் Answer:**
"Terraform CI/CD-ல் secrets மூன்று levels-ல் handle:

1. **CI Authentication:** OIDC — stored credentials இல்லை. Short-lived tokens.

2. **Terraform variables:** CI system-ன் secret store-ல் store. Environment variables-ஆக inject. Code-ல் ஒருபோதும் இல்லை.

3. **State file:** State-ல் sensitive values plaintext-ல் உள்ளன — state backend encryption + strict access controls கட்டாயம்.

4. **Outputs:** `sensitive = true` mark, CI logs-ல் log ஆகாது.

Golden rule: secrets exactly இரண்டு இடங்களில் மட்டுமே — secrets manager + running infrastructure. Git-ல் ஒருபோதும் இல்லை."

---

## ✅ Self-Check

### Knowledge Verification

- [ ] Can you draw the plan-on-PR/apply-on-merge pattern from memory?
- [ ] Can you explain OIDC in 3 sentences?
- [ ] Can you write a basic GitHub Actions Terraform workflow?
- [ ] Do you know the difference between Atlantis and Jenkins approaches?
- [ ] Can you explain drift detection implementation?
- [ ] Can you describe the EB pipeline architecture?

### Interview Readiness

- [ ] Can you design a pipeline for 50 developers on a whiteboard?
- [ ] Can you explain state locking in CI/CD?
- [ ] Can you describe rollback strategies?
- [ ] Can you talk about secret management in pipelines?

### Command Competence

```bash
# Core CI/CD commands you should know:
terraform init -input=false -backend-config=env.hcl
terraform plan -out=tfplan -no-color -input=false
terraform apply -auto-approve -input=false tfplan
terraform plan -detailed-exitcode  # Exit 2 = drift detected
terraform force-unlock <LOCK_ID>
```

---

## 📋 Quick Reference Card

```
┌────────────────────────────────────────────────────────────────┐
│           TERRAFORM CI/CD — CHEAT SHEET                         │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  GOLDEN PATTERN:                                                │
│    PR → init → validate → scan → plan → comment                │
│    Merge → init → plan → [approve] → apply → notify            │
│                                                                  │
│  AUTH:                                                          │
│    OIDC > Service Account Key (always)                          │
│    GCP: Workload Identity Federation                            │
│    AWS: OIDC Provider + IAM Role                                │
│                                                                  │
│  STATE LOCKING:                                                 │
│    Backend-level: GCS .tflock / S3 DynamoDB                     │
│    CI-level: concurrency groups / disableConcurrentBuilds       │
│                                                                  │
│  DRIFT DETECTION:                                               │
│    terraform plan -detailed-exitcode                            │
│    Exit 0 = no changes, Exit 2 = drift detected                │
│    Schedule: every 4 hours minimum                              │
│                                                                  │
│  ROLLBACK:                                                      │
│    Code revert PR → CI applies previous state                   │
│    State recovery → restore from versioned bucket               │
│                                                                  │
│  SECRETS:                                                       │
│    Auth: OIDC (no stored keys)                                  │
│    Variables: CI secret store → TF_VAR_*                        │
│    State: Encrypted backend, restricted access                  │
│                                                                  │
│  ARCHITECT MINDSET:                                             │
│    "No developer ever runs apply locally"                       │
│    "Plan is cheap, apply is expensive — review before apply"    │
│    "Isolation per environment, not just per workspace"          │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```
