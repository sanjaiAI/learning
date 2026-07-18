┌─────────────────────────────────────────────────────────────────────────────┐
│                        JENKINS BUILD FAILS                                  │
│  Pipeline: BRI/playground/sass273491-sanjaivasan_ss/mcp-integration-test    │
│  Build bot: deulmhu25                                                       │
└─────────────┬───────────────────────────────────────────────────────────────┘
              │
              │  post { failure { ... } }  triggers
              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 1 — Load credentials from Jenkins Credential Store                    │
│                                                                             │
│  withCredentials([usernamePassword(                                         │
│      credentialsId: 'aiops-keycloak-creds',                                 │
│      usernameVariable: 'KC_CLIENT_ID',       ← jenkins-webhook              │
│      passwordVariable: 'KC_CLIENT_SECRET'    ← jenkins-webhook-secret-2026  │
│  )])                                                                        │
└─────────────┬───────────────────────────────────────────────────────────────┘
              │
              │  curl POST (client_credentials grant)
              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 2 — Get JWT from Keycloak                                             │
│                                                                             │
│  POST http://b50025v:8180/realms/mcp/protocol/openid-connect/token          │
│    grant_type=client_credentials                                            │
│    client_id=jenkins-webhook                                                │
│    client_secret=jenkins-webhook-secret-2026                                │
│                                                                             │
│  Keycloak returns JWT with claims:                                          │
│    aud: ["mcp-aiops", "account"]    ← injected by aiops-audience mapper     │
│    iss: http://b50025v:8180/realms/mcp                                      │
│    azp: jenkins-webhook                                                     │
│    client_id: jenkins-webhook                                               │
│    exp: <now + 3600s>                                                       │
└─────────────┬───────────────────────────────────────────────────────────────┘
              │
              │  curl POST with Authorization: Bearer <jwt>
              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 3 — Send failure to AIOPS webhook                                     │
│                                                                             │
│  POST http://inblr1pr001:3080/api/webhook                                   │
│  Headers:                                                                   │
│    Content-Type: application/json                                           │
│    Authorization: Bearer <jwt>                                              │
│  Body:                                                                      │
│    {                                                                        │
│      "job_url": "https://skilfish-jenkins.../job/.../5/",                   │
│      "job_name": "BRI/playground/.../mcp-integration-test",                 │
│      "build_number": 5,                                                     │
│      "status": "FAILURE",                                                   │
│      "triggered_by": "webhook"                                              │
│    }                                                                        │
└─────────────┬───────────────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 4 — AIOPS Agent validates JWT (auth.py → require_api_key)             │
│                                                                             │
│  1. Extracts Bearer token from Authorization header                         │
│  2. Detects it's a JWT (has 2 dots)                                         │
│  3. Fetches Keycloak public key from JWKS endpoint:                         │
│     http://b50025v:8180/realms/mcp/protocol/openid-connect/certs            │
│  4. Validates:                                                              │
│     ✓ RS256 signature (using JWKS public key)                               │
│     ✓ iss == http://b50025v:8180/realms/mcp                                 │
│     ✓ aud contains "mcp-aiops"       ← why the audience mapper matters     │
│     ✓ exp > now (not expired)                                               │
│  5. Auth passes → request proceeds                                          │
└─────────────┬───────────────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 5 — AIOPS runs full LangGraph pipeline (18 nodes)                     │
│                                                                             │
│  webhook() → analyze() → LangGraph pipeline:                                │
│    parse → tokenize → pre_filter → analyst_council → judge → knowledge      │
│    → publish → mcp_remediate → continuous_learn → END                       │
│                                                                             │
│  If mcp_remediate fires (KEDB match + confidence ≥ 0.5):                    │
│    → Gets its own JWT from Keycloak using mcp-aiops client                  │
│    → Calls MCP PROD REST API at b50025v:3457/api/tools/call                 │
│    → Removes block file, retriggers build                                   │
└─────────────────────────────────────────────────────────────────────────────┘
