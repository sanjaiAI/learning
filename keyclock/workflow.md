# Jenkins Build Failure Flow

## Overview

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                        JENKINS BUILD FAILS                                  │
│  Pipeline: BRI/playground/sass273491-sanjaivasan_ss/mcp-integration-test    │
│  Build bot: deulmhu25                                                       │
└─────────────┬───────────────────────────────────────────────────────────────┘
              │
              ▼
```

---

## Step 1 – Load Credentials from Jenkins

The failure notification is triggered from the Jenkins `post { failure { ... } }` block.

Jenkins retrieves the Keycloak client credentials from the Jenkins Credential Store.

```groovy
withCredentials([
    usernamePassword(
        credentialsId: 'aiops-keycloak-creds',
        usernameVariable: 'KC_CLIENT_ID',
        passwordVariable: 'KC_CLIENT_SECRET'
    )
])
```

| Variable | Value |
|----------|-------|
| Client ID | `jenkins-webhook` |
| Client Secret | `jenkins-webhook-secret-2026` |

---

## Step 2 – Obtain JWT from Keycloak

Jenkins authenticates using the **OAuth2 Client Credentials** flow.

### Request

```
POST http://b50025v:8180/realms/mcp/protocol/openid-connect/token
```

Form data:

```text
grant_type=client_credentials
client_id=jenkins-webhook
client_secret=jenkins-webhook-secret-2026
```

### Keycloak returns

A signed JWT containing claims similar to:

```yaml
iss: http://b50025v:8180/realms/mcp
aud:
  - mcp-aiops
  - account
azp: jenkins-webhook
client_id: jenkins-webhook
exp: <current time + 3600 seconds>
```

> The `aud` claim contains **mcp-aiops** because of the configured **Audience Mapper** in Keycloak.

---

## Step 3 – Send Failure Notification to AIOPS

Jenkins sends the build failure to the AIOPS webhook.

### Request

```
POST http://inblr1pr001:3080/api/webhook
```

Headers

```http
Content-Type: application/json
Authorization: Bearer <JWT>
```

Payload

```json
{
  "job_url": "https://skilfish-jenkins.../job/.../5/",
  "job_name": "BRI/playground/.../mcp-integration-test",
  "build_number": 5,
  "status": "FAILURE",
  "triggered_by": "webhook"
}
```

---

## Step 4 – AIOPS Validates the JWT

The webhook reaches the authentication layer (`auth.py → require_api_key`).

Validation flow:

1. Extract the Bearer token.
2. Detect that it is a JWT (contains two dots).
3. Retrieve the Keycloak public key (JWKS):

```
http://b50025v:8180/realms/mcp/protocol/openid-connect/certs
```

4. Validate:

- ✅ RS256 signature
- ✅ Issuer (`iss`)
- ✅ Audience contains `mcp-aiops`
- ✅ Token has not expired

If all validations succeed, the request is authorized.

---

## Step 5 – Execute the AIOPS LangGraph Pipeline

The webhook invokes the complete LangGraph workflow.

```text
webhook()
    │
    ▼
analyze()
    │
    ▼
parse
    │
    ▼
tokenize
    │
    ▼
pre_filter
    │
    ▼
analyst_council
    │
    ▼
judge
    │
    ▼
knowledge
    │
    ▼
publish
    │
    ▼
mcp_remediate
    │
    ▼
continuous_learn
    │
    ▼
END
```

---

## Automatic MCP Remediation

If the remediation node determines:

- A KEDB match exists
- Confidence ≥ **0.5**

then it performs the following steps:

1. Authenticate with Keycloak using the **mcp-aiops** client.
2. Obtain a new JWT.
3. Invoke the MCP Production REST API.

```
POST http://b50025v:3457/api/tools/call
```

The MCP server then performs the remediation, for example:

- Remove the block file
- Retrigger the Jenkins build

---

# End-to-End Flow

```text
Jenkins Build Failure
        │
        ▼
Load Jenkins Credentials
        │
        ▼
Request JWT from Keycloak
        │
        ▼
POST Failure Event to AIOPS Webhook
        │
        ▼
JWT Validation (JWKS)
        │
        ▼
LangGraph Analysis Pipeline
        │
        ▼
KEDB Match?
   │             │
   │ No          │ Yes
   ▼             ▼
Publish      Authenticate to Keycloak
Results             │
                    ▼
             Call MCP REST API
                    │
                    ▼
           Execute Auto Remediation
                    │
                    ▼
          Retrigger Jenkins Build
```
