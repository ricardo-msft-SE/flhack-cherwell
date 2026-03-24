# Cross-Tenant Azure AI Foundry → NWRDC Azure OpenAI

**Pattern**: Agency AI Foundry agent calls NWRDC-hosted Azure OpenAI models using Workload Identity Federation — no secrets, no API keys, no Lighthouse, no network peering.

---

## 1. Architecture Overview

### Topology

```
┌──────────────────────────────────────────────────────┐
│  AGENCY TENANT                                        │
│                                                       │
│  ┌────────────────────────────────────────────┐      │
│  │  Azure AI Foundry Resource (AIServices)     │      │
│  │  kind=AIServices (CognitiveServices/accts)  │      │
│  │  └── Foundry Project                        │      │
│  │       ├── User-Assigned MI (UAMI)           │      │
│  │       ├── AI Agents (azure-ai-projects SDK) │      │
│  │       ├── Azure AI Search connection        │      │
│  │       │   (same-tenant, Entra ID RBAC)      │      │
│  │       └── Threads, Evaluations, Storage     │      │
│  └────────────────────────────────────────────┘      │
│                              │                        │
│              1. GET IMDS token                        │
│              audience=api://AzureADTokenExchange      │
└──────────────────────────────────────────────────────┘
                               │
              2. POST token exchange to NWRDC
              token endpoint (client_credentials +
              client_assertion)
                               │
                               ▼
┌──────────────────────────────────────────────────────┐
│  NWRDC TENANT                                         │
│                                                       │
│  ┌───────────────────────────────────┐               │
│  │  App Registration (single-tenant) │               │
│  │  "agency-foundry-model-accessor"  │               │
│  │  Federated Credential:            │               │
│  │    issuer: Agency tenant OIDC     │               │
│  │    subject: Agency UAMI obj ID    │               │
│  │    audience: api://AzureADToken   │               │
│  │             Exchange              │               │
│  └───────────────────────────────────┘               │
│              │                                        │
│              3. Issues token for NWRDC SP             │
│              │                                        │
│              ▼                                        │
│  ┌────────────────────────────────────────────┐      │
│  │  Azure OpenAI Resource                      │      │
│  │  ├── gpt-4o (Standard, 100K TPM)           │      │
│  │  └── RBAC: Cognitive Services OpenAI User  │      │
│  │         → App Registration's SP object ID  │      │
│  └────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────┘
```

### Control Plane vs. Data Plane

| Layer | URI pattern | Owner | Auth |
|---|---|---|---|
| Control plane (ARM) | `management.azure.com/...` | Both admins | Azure RBAC via Entra |
| AOAI data plane | `{resource}.openai.azure.com/openai/...` | NWRDC | Token from NWRDC tenant via WIF |
| Foundry agent plane | `{project}.services.ai.azure.com/api/...` | Agency | Agency Entra ID (UAMI) |
| AI Search data plane | `{search}.search.windows.net/...` | Agency | Agency Entra ID (UAMI) |

### Identity Flow (per inference call)

1. Agency agent compute (bearing UAMI) calls IMDS: requests JWT with audience `api://AzureADTokenExchange`
2. Code posts that JWT as a `client_assertion` to NWRDC's Entra token endpoint, identifying as the NWRDC app registration
3. NWRDC Entra validates assertion against the configured federated credential (issuer + subject match)
4. Returns a bearer token for the NWRDC app's service principal, scoped to `https://cognitiveservices.azure.com/.default`
5. That token is used in the `Authorization: Bearer` header against the NWRDC AOAI endpoint
6. NWRDC AOAI validates RBAC: NWRDC SP has `Cognitive Services OpenAI User` → inference proceeds

### Why Lighthouse Does NOT Solve This

Lighthouse has two independent blocking constraints, either of which alone prevents it:

| Constraint | Detail |
|---|---|
| Data plane exclusion | Lighthouse only delegates requests to `management.azure.com`. AOAI inference hits `{resource}.openai.azure.com` — a resource-instance URI that Lighthouse cannot reach |
| DataActions exclusion | Lighthouse explicitly cannot delegate roles containing `DataActions`. `Cognitive Services OpenAI User` contains `DataActions` (chat completions, embeddings, etc.) |

Lighthouse can delegate ARM operations at NWRDC (e.g., deploying models via Bicep), but **cannot delegate inference**.

---

## 2. Responsibility Matrix

| Task | NWRDC Admin | Agency Admin | Developer | Required? |
|---|---|---|---|---|
| Deploy Azure OpenAI resource | ✅ | | | Required |
| Deploy model in NWRDC | ✅ | | | Required |
| Create app registration (NWRDC) | ✅ | | | Required |
| Configure federated credential | ✅ (after receiving UAMI obj ID) | | | Required |
| Assign `Cognitive Services OpenAI User` RBAC | ✅ | | | Required |
| Share: tenant ID, app client ID, AOAI endpoint | ✅ → Agency | | | Required |
| Create UAMI in Agency tenant | | ✅ | | Required |
| Assign UAMI to Foundry AIServices resource | | ✅ | | Required |
| Share UAMI service principal Object ID | | ✅ → NWRDC | | Required |
| Assign RBAC for AI Search to UAMI | | ✅ | | Required |
| Add NWRDC AOAI as Foundry project connection | | ✅ (portal/Bicep) | | Optional (for UI agent config) |
| Implement cross-tenant credential in code | | | ✅ | Required |
| Implement agent with tools | | | ✅ | Required |
| Set `disableLocalAuth: true` on AOAI | ✅ | | | Recommended |
| Enable diagnostic logs on AOAI | ✅ | | | Recommended |
| Configure Conditional Access for NWRDC app | ✅ | | | Recommended |

---

## 3. Step-by-Step Implementation

### Phase A — NWRDC Admin

**Prerequisites**: Logged into Azure CLI as NWRDC tenant owner/contributor. Replace all `<...>` placeholders.

---

#### A1. Deploy Azure OpenAI Resource and Model (Bicep)

**File**: `nwrdc-aoai.bicep` | **Tenant**: NWRDC

```bicep
// TENANT CONTEXT: NWRDC
// Run: az deployment group create --resource-group <rg> --template-file nwrdc-aoai.bicep --parameters ...

@description('Name for the Azure OpenAI account. Must be globally unique.')
param aoaiName string

@description('Azure region for deployment.')
param location string = resourceGroup().location

@description('GPT model name to deploy, e.g. gpt-4o')
param modelName string = 'gpt-4o'

@description('Model version.')
param modelVersion string = '2024-11-20'

@description('Tokens-per-minute capacity in thousands (e.g. 100 = 100K TPM).')
param tpmCapacity int = 100

@description('Object ID of the NWRDC App Registration service principal. Set after A2 is complete.')
param agencyAppPrincipalObjectId string

var cogServicesOpenAIUserRoleId = '5e0bd9bd-7b93-4f28-af87-19fc36ad61bd'

// ── Azure OpenAI Account ──────────────────────────────────────────────────────
resource aoai 'Microsoft.CognitiveServices/accounts@2023-05-01' = {
  name: aoaiName
  location: location
  kind: 'OpenAI'
  sku: {
    name: 'S0'
  }
  properties: {
    customSubDomainName: aoaiName        // Required for Entra ID token audience validation
    publicNetworkAccess: 'Enabled'
    disableLocalAuth: true               // Enforce Entra ID — no API key fallback
    networkAcls: {
      defaultAction: 'Allow'            // Adjust to 'Deny' + IP allowlist for higher security
    }
  }
}

// ── Model Deployment ──────────────────────────────────────────────────────────
resource modelDeployment 'Microsoft.CognitiveServices/accounts/deployments@2023-05-01' = {
  parent: aoai
  name: modelName
  sku: {
    name: 'Standard'
    capacity: tpmCapacity
  }
  properties: {
    model: {
      format: 'OpenAI'
      name: modelName
      version: modelVersion
    }
    versionUpgradeOption: 'OnceNewDefaultVersionAvailable'
  }
}

// ── RBAC: Cognitive Services OpenAI User on the AOAI resource ─────────────────
// Scoped to specific resource (not RG or subscription) — least privilege
resource rbacAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(aoai.id, agencyAppPrincipalObjectId, cogServicesOpenAIUserRoleId)
  scope: aoai
  properties: {
    roleDefinitionId: subscriptionResourceId(
      'Microsoft.Authorization/roleDefinitions',
      cogServicesOpenAIUserRoleId
    )
    principalId: agencyAppPrincipalObjectId
    principalType: 'ServicePrincipal'  // Prevents lookup delay; prevents accidental user assignment
  }
}

// ── Outputs ────────────────────────────────────────────��──────────────────────
output aoaiEndpoint string = aoai.properties.endpoint
output aoaiResourceId string = aoai.id
output modelDeploymentName string = modelDeployment.name
```

**Deployment command** (NWRDC tenant):

```bash
# TENANT CONTEXT: NWRDC
az login --tenant <nwrdc-tenant-id>

az deployment group create \
  --resource-group nwrdc-aoai-rg \
  --template-file nwrdc-aoai.bicep \
  --parameters \
      aoaiName=nwrdc-aoai-prod \
      modelName=gpt-4o \
      modelVersion=2024-11-20 \
      tpmCapacity=100 \
      agencyAppPrincipalObjectId=<fill-in-after-A2>
```

> **Note**: Deploy once without `agencyAppPrincipalObjectId` to get the resource created, then re-run with the value from A2 to apply RBAC. Or do both in a single deployment once A2 is complete.

---

#### A2. Create App Registration and Federated Credential (CLI)

**Tenant**: NWRDC | **Role required**: Application Administrator or Global Administrator

```bash
# TENANT CONTEXT: NWRDC

NWRDC_TENANT_ID="<nwrdc-tenant-id>"
AGENCY_TENANT_ID="<agency-tenant-id>"
AGENCY_UAMI_OBJECT_ID="<agency-uami-service-principal-object-id>"  # From Agency admin (B1 output)
APP_DISPLAY_NAME="agency-foundry-model-accessor"

# 1. Create the app registration (single-tenant — no cross-tenant consent required)
APP_ID=$(az ad app create \
  --display-name "$APP_DISPLAY_NAME" \
  --sign-in-audience AzureADMyOrg \
  --query appId -o tsv)

echo "App Registration Client ID: $APP_ID"

# 2. Create the service principal for this app in NWRDC tenant
SP_OBJECT_ID=$(az ad sp create \
  --id "$APP_ID" \
  --query id -o tsv)

echo "Service Principal Object ID (NWRDC): $SP_OBJECT_ID"
# ↑ This value goes into the Bicep 'agencyAppPrincipalObjectId' parameter

# 3. Add the federated identity credential
#    issuer: Agency tenant's OIDC endpoint (v2.0 suffix required)
#    subject: Agency UAMI's service principal object ID (NOT the client ID / resource ID)
#    audience: standard cross-tenant WIF audience
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters "{
    \"name\": \"agency-foundry-uami-fed\",
    \"issuer\": \"https://login.microsoftonline.com/${AGENCY_TENANT_ID}/v2.0\",
    \"subject\": \"${AGENCY_UAMI_OBJECT_ID}\",
    \"description\": \"Allows Agency Foundry UAMI to exchange tokens for NWRDC AOAI access\",
    \"audiences\": [\"api://AzureADTokenExchange\"]
  }"

echo "---"
echo "Share with Agency team:"
echo "  NWRDC Tenant ID:      $NWRDC_TENANT_ID"
echo "  App Registration ID:  $APP_ID"
echo "  AOAI Endpoint:        (from Bicep output)"
echo "  Model Deployment:     gpt-4o"
```

---

#### A3. Assign RBAC via CLI (alternative to Bicep)

```bash
# TENANT CONTEXT: NWRDC
# Use this if applying RBAC separately from the Bicep deployment

AOAI_RESOURCE_ID="/subscriptions/<nwrdc-subscription-id>/resourceGroups/nwrdc-aoai-rg/providers/Microsoft.CognitiveServices/accounts/nwrdc-aoai-prod"

az role assignment create \
  --assignee-object-id "$SP_OBJECT_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "Cognitive Services OpenAI User" \
  --scope "$AOAI_RESOURCE_ID"
```

---

#### A4. NWRDC Validation

```bash
# TENANT CONTEXT: NWRDC

# Confirm federated credential is configured correctly
az ad app federated-credential list --id "$APP_ID" -o table

# Confirm RBAC assignment
az role assignment list \
  --assignee "$SP_OBJECT_ID" \
  --scope "$AOAI_RESOURCE_ID" \
  -o table

# Confirm disableLocalAuth is set
az cognitiveservices account show \
  --name nwrdc-aoai-prod \
  --resource-group nwrdc-aoai-rg \
  --query "properties.disableLocalAuth"
# Expected: true

# Confirm model deployment is ready
az cognitiveservices account deployment show \
  --name nwrdc-aoai-prod \
  --resource-group nwrdc-aoai-rg \
  --deployment-name gpt-4o \
  --query "properties.provisioningState"
# Expected: "Succeeded"
```

---

### Phase B — Agency Admin

**Prerequisites**: Logged into Azure CLI as Agency tenant contributor. Agency AI Foundry **Project** (new experience — backed by `Microsoft.CognitiveServices/accounts` with `kind=AIServices`) is already provisioned. If you are on the classic Hub+Project model (`Microsoft.MachineLearningServices/workspaces`), the B2 and B4 CLI commands differ — see the note in each step.

---

#### B1. Create the User-Assigned Managed Identity

```bash
# TENANT CONTEXT: AGENCY

AGENCY_RG="agency-foundry-rg"
AGENCY_LOCATION="eastus"
UAMI_NAME="foundry-crosstenantmodel-mi"

az identity create \
  --name "$UAMI_NAME" \
  --resource-group "$AGENCY_RG" \
  --location "$AGENCY_LOCATION"

# Resource ID — used to attach the UAMI to compute resources
UAMI_RESOURCE_ID=$(az identity show \
  --name "$UAMI_NAME" \
  --resource-group "$AGENCY_RG" \
  --query id -o tsv)

# Client ID — used in ManagedIdentityCredential in developer code
UAMI_CLIENT_ID=$(az identity show \
  --name "$UAMI_NAME" \
  --resource-group "$AGENCY_RG" \
  --query clientId -o tsv)

# CRITICAL: principalId is the service principal Object ID — this is the 'subject'
# for the federated credential in NWRDC. NOT the clientId and NOT the resource objectId.
UAMI_SP_OBJECT_ID=$(az identity show \
  --name "$UAMI_NAME" \
  --resource-group "$AGENCY_RG" \
  --query principalId -o tsv)

echo "Share with NWRDC admin:"
echo "  UAMI Service Principal Object ID: $UAMI_SP_OBJECT_ID"
echo "  Agency Tenant ID: $(az account show --query tenantId -o tsv)"

echo "Store for developer use:"
echo "  UAMI Client ID (for ManagedIdentityCredential): $UAMI_CLIENT_ID"
echo "  UAMI Resource ID: $UAMI_RESOURCE_ID"
```

---

#### B2. Assign the UAMI to the Foundry Resource

In the new Foundry experience, the backing resource is a **Cognitive Services account with `kind=AIServices`** — not an ML workspace Hub. Identity is assigned directly to that account.

```bash
# TENANT CONTEXT: AGENCY
# New Foundry experience: resource type is Microsoft.CognitiveServices/accounts (kind=AIServices)
# The Foundry resource name is the AI Services account name, visible in the Azure portal
# under the AI Foundry project → Settings → Resource details.

FOUNDRY_RESOURCE_NAME="agency-ai-foundry"   # The AIServices account name
FOUNDRY_RG="agency-foundry-rg"

# Assign the UAMI to the Foundry AIServices account
az cognitiveservices account identity assign \
  --name "$FOUNDRY_RESOURCE_NAME" \
  --resource-group "$FOUNDRY_RG" \
  --user-assigned "$UAMI_RESOURCE_ID"

# Verify the assignment
az cognitiveservices account show \
  --name "$FOUNDRY_RESOURCE_NAME" \
  --resource-group "$FOUNDRY_RG" \
  --query "identity.userAssignedIdentities" -o table
# Expected: entry showing $UAMI_RESOURCE_ID with a clientId and principalId
```

> **Note**: If your agents run on a separate compute host (Container Apps, AKS, VM), assign the UAMI there too:

```bash
# For Container Apps agent compute (if applicable)
az containerapp identity assign \
  --name <containerapp-name> \
  --resource-group "$FOUNDRY_RG" \
  --user-assigned "$UAMI_RESOURCE_ID"
```

---

#### B3. Assign RBAC for Agency Resources to the UAMI

```bash
# TENANT CONTEXT: AGENCY

AI_SEARCH_RESOURCE_ID="/subscriptions/<agency-sub>/resourceGroups/$AGENCY_RG/providers/Microsoft.Search/searchServices/<search-name>"

az role assignment create \
  --assignee-object-id "$UAMI_SP_OBJECT_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "Search Index Data Reader" \
  --scope "$AI_SEARCH_RESOURCE_ID"

az role assignment create \
  --assignee-object-id "$UAMI_SP_OBJECT_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "Search Service Contributor" \
  --scope "$AI_SEARCH_RESOURCE_ID"
```

---

#### B4. Add NWRDC AOAI as a Connected Resource (Optional — for UI agent builder)

> Registers the NWRDC AOAI endpoint in the Foundry portal UI so it appears in the agent builder model picker. Auth for programmatic calls is handled by developer code (Phase C) — the new Foundry Project experience does not perform cross-tenant WIF token exchange natively through connections.

In the new Foundry experience, the `az ml connection` command (Azure ML CLI extension) **does not apply** — connections are managed through the Foundry REST API or the portal.

**Option A — Portal (recommended for one-time setup)**:
1. Open [ai.azure.com](https://ai.azure.com) → your Agency project
2. Navigate to **Management** → **Connected resources** → **+ New connection**
3. Select **Azure OpenAI**
4. Enter the NWRDC AOAI endpoint: `https://nwrdc-aoai-prod.openai.azure.com/`
5. Set authentication to **API Key**, enter `PLACEHOLDER_REPLACED_IN_CODE` as the key
6. Name the connection `nwrdc-aoai-connection` and save

> `disableLocalAuth: true` on the NWRDC AOAI resource ensures this placeholder key is **rejected server-side** even if accidentally used. Real inference is always routed through the WIF credential in Phase C code.

**Option B — REST API (for automation/IaC)**:

```bash
# TENANT CONTEXT: AGENCY
# Uses the Foundry Project REST API to create the connection
# Requires: az login as Agency contributor; jq installed

FOUNDRY_RESOURCE_NAME="agency-ai-foundry"
FOUNDRY_PROJECT_NAME="agency-project"
FOUNDRY_RG="agency-foundry-rg"
AGENCY_SUB_ID=$(az account show --query id -o tsv)

ACCESS_TOKEN=$(az account get-access-token \
  --resource https://management.azure.com \
  --query accessToken -o tsv)

curl -s -X PUT \
  "https://management.azure.com/subscriptions/${AGENCY_SUB_ID}/resourceGroups/${FOUNDRY_RG}/providers/Microsoft.CognitiveServices/accounts/${FOUNDRY_RESOURCE_NAME}/projects/${FOUNDRY_PROJECT_NAME}/connections/nwrdc-aoai-connection?api-version=2025-04-01-preview" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "properties": {
      "category": "AzureOpenAI",
      "target": "https://nwrdc-aoai-prod.openai.azure.com/",
      "authType": "ApiKey",
      "credentials": {
        "key": "PLACEHOLDER_REPLACED_IN_CODE"
      },
      "metadata": {
        "ApiType": "azure",
        "ApiVersion": "2024-10-21"
      }
    }
  }'
```

---

### Phase C — Developer

#### C1. Environment Configuration

Store in Key Vault, Container App secrets, or environment variables. Never hardcode.

```bash
# Values from NWRDC admin (A2 outputs)
NWRDC_TENANT_ID=<nwrdc-tenant-id>
NWRDC_APP_CLIENT_ID=<app-registration-client-id>
NWRDC_AOAI_ENDPOINT=https://nwrdc-aoai-prod.openai.azure.com/
NWRDC_MODEL_DEPLOYMENT=gpt-4o

# Values from Agency admin (B1 output)
AGENCY_UAMI_CLIENT_ID=<uami-client-id>

# Agency Foundry project endpoint (new experience: AIServices resource + project name)
AGENCY_FOUNDRY_ENDPOINT=https://<aiservices-resource-name>.services.ai.azure.com/api/projects/<project-name>

# Agency AI Search
AGENCY_SEARCH_ENDPOINT=https://<search-name>.search.windows.net
AGENCY_SEARCH_INDEX=my-knowledge-index
```

---

#### C2. Cross-Tenant Credential Module

**File**: `cross_tenant_credential.py` | **Runs in**: Agency tenant compute bearing the UAMI

```python
# Implements WIF token exchange: UAMI assertion → NWRDC bearer token
# No secrets stored anywhere. Token refresh is handled by the SDK.

import os
from azure.identity import ManagedIdentityCredential, ClientAssertionCredential


def build_cross_tenant_credential(
    uami_client_id: str,
    nwrdc_tenant_id: str,
    nwrdc_app_client_id: str,
) -> ClientAssertionCredential:
    """
    Returns a credential that authenticates as the NWRDC app registration's
    service principal by using the UAMI's OIDC JWT as a federated assertion.

    Flow:
      1. UAMI requests a JWT from IMDS with audience 'api://AzureADTokenExchange'
      2. That JWT is presented as a client_assertion to the NWRDC token endpoint
      3. NWRDC validates it against the configured federated identity credential
      4. Returns a bearer token scoped to cognitiveservices.azure.com in NWRDC
    """
    mi_credential = ManagedIdentityCredential(client_id=uami_client_id)

    def get_uami_assertion() -> str:
        """
        Called by ClientAssertionCredential each time a new token is needed.
        Returns the UAMI's OIDC JWT for the WIF exchange audience.
        """
        token = mi_credential.get_token("api://AzureADTokenExchange")
        return token.token

    return ClientAssertionCredential(
        tenant_id=nwrdc_tenant_id,
        client_id=nwrdc_app_client_id,
        func=get_uami_assertion,
    )
```

---

#### C3. Agent Implementation

**File**: `agent.py` | **Runs in**: Agency tenant compute bearing the UAMI

```python
# Agent management (tools, threads, eval) → Agency Foundry project (same-tenant)
# Model inference → NWRDC Azure OpenAI via WIF cross-tenant credential

import os
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import (
    AzureAISearchTool,
    AzureAISearchQueryType,
    ConnectionType,
)
from openai import AzureOpenAI

from cross_tenant_credential import build_cross_tenant_credential


# ── Cross-tenant credential for NWRDC AOAI ───────────────────────────────────
nwrdc_credential = build_cross_tenant_credential(
    uami_client_id=os.environ["AGENCY_UAMI_CLIENT_ID"],
    nwrdc_tenant_id=os.environ["NWRDC_TENANT_ID"],
    nwrdc_app_client_id=os.environ["NWRDC_APP_CLIENT_ID"],
)

# ── AzureOpenAI client pointing at NWRDC ─────────────────────────────────────
# Uses the WIF-exchanged NWRDC token for all inference calls.
nwrdc_openai_client = AzureOpenAI(
    base_url=f"{os.environ['NWRDC_AOAI_ENDPOINT']}openai/v1/",
    azure_ad_token_provider=get_bearer_token_provider(
        nwrdc_credential,
        "https://cognitiveservices.azure.com/.default",
    ),
    api_version="2024-10-21",
)

# ── Agency Foundry project client (same-tenant, standard credential) ──────────
# Used for: agent registration, thread management, tool connections,
#           AI Search integration, evaluations, telemetry.
agency_foundry_client = AIProjectClient(
    endpoint=os.environ["AGENCY_FOUNDRY_ENDPOINT"],
    credential=DefaultAzureCredential(
        managed_identity_client_id=os.environ["AGENCY_UAMI_CLIENT_ID"]
    ),
)

# ── Resolve AI Search connection from Agency Foundry project ─────────────────
search_connection = agency_foundry_client.connections.get_default(
    connection_type=ConnectionType.AZURE_AI_SEARCH
)


def create_or_get_agent(agent_name: str = "agency-cross-tenant-agent"):
    """
    Creates the agent definition in the Agency Foundry project.
    The agent is stored in Agency tenant.
    At runtime, inference calls go to NWRDC AOAI via the WIF credential.
    """
    agents = agency_foundry_client.agents.list_agents()
    for existing in agents.data:
        if existing.name == agent_name:
            print(f"Reusing existing agent: {existing.id}")
            return existing

    search_tool = AzureAISearchTool(
        connection_id=search_connection.id,
        index_name=os.environ["AGENCY_SEARCH_INDEX"],
        query_type=AzureAISearchQueryType.VECTOR_SEMANTIC_HYBRID,
    )

    agent = agency_foundry_client.agents.create_agent(
        model=os.environ["NWRDC_MODEL_DEPLOYMENT"],  # "gpt-4o"
        name=agent_name,
        instructions=(
            "You are a helpful assistant with access to the agency knowledge base. "
            "Use the search tool to retrieve relevant documents before answering. "
            "Always cite the source document title and url in your response."
        ),
        tools=search_tool.definitions,
        tool_resources=search_tool.resources,
    )
    print(f"Created agent: {agent.id}")
    return agent


def run_agent_turn(agent_id: str, thread_id: str | None, user_message: str) -> dict:
    """
    Executes a single agent turn.
    Thread management and tool calls use Agency Foundry (same-tenant).
    Model completion is routed to NWRDC AOAI via the WIF credential.
    """
    if thread_id is None:
        thread = agency_foundry_client.agents.create_thread()
        thread_id = thread.id

    agency_foundry_client.agents.create_message(
        thread_id=thread_id,
        role="user",
        content=user_message,
    )

    run = agency_foundry_client.agents.create_and_process_run(
        thread_id=thread_id,
        agent_id=agent_id,
    )

    if run.status == "failed":
        raise RuntimeError(f"Agent run failed: {run.last_error}")

    messages = agency_foundry_client.agents.list_messages(thread_id=thread_id)
    last_message = next(m for m in messages.data if m.role == "assistant")

    return {
        "thread_id": thread_id,
        "run_id": run.id,
        "response": last_message.content[0].text.value,
    }


if __name__ == "__main__":
    agent = create_or_get_agent()

    result = run_agent_turn(
        agent_id=agent.id,
        thread_id=None,
        user_message="What are the data retention policies for citizen records?",
    )

    print(f"Thread: {result['thread_id']}")
    print(f"Response:\n{result['response']}")
```

---

#### C4. Token Exchange Validation Script

**File**: `validate_cross_tenant.py` | Run this on target compute before deploying the agent.

```python
import os
from azure.identity import ManagedIdentityCredential, get_bearer_token_provider
from openai import AzureOpenAI
from cross_tenant_credential import build_cross_tenant_credential


def validate():
    print("Step 1: Acquire UAMI assertion from IMDS...")
    mi = ManagedIdentityCredential(client_id=os.environ["AGENCY_UAMI_CLIENT_ID"])
    assertion = mi.get_token("api://AzureADTokenExchange")
    print(f"  ✓ IMDS token acquired (exp: {assertion.expires_on})")

    print("Step 2: Exchange assertion for NWRDC bearer token...")
    nwrdc_cred = build_cross_tenant_credential(
        uami_client_id=os.environ["AGENCY_UAMI_CLIENT_ID"],
        nwrdc_tenant_id=os.environ["NWRDC_TENANT_ID"],
        nwrdc_app_client_id=os.environ["NWRDC_APP_CLIENT_ID"],
    )
    nwrdc_token = nwrdc_cred.get_token("https://cognitiveservices.azure.com/.default")
    print(f"  ✓ NWRDC bearer token acquired (exp: {nwrdc_token.expires_on})")

    print("Step 3: Call NWRDC AOAI with the bearer token...")
    client = AzureOpenAI(
        base_url=f"{os.environ['NWRDC_AOAI_ENDPOINT']}openai/v1/",
        azure_ad_token_provider=get_bearer_token_provider(
            nwrdc_cred, "https://cognitiveservices.azure.com/.default"
        ),
        api_version="2024-10-21",
    )
    response = client.chat.completions.create(
        model=os.environ["NWRDC_MODEL_DEPLOYMENT"],
        messages=[{"role": "user", "content": "Reply with exactly: CROSS_TENANT_OK"}],
        max_tokens=10,
    )
    reply = response.choices[0].message.content.strip()
    assert "CROSS_TENANT_OK" in reply, f"Unexpected reply: {reply}"
    print(f"  ✓ NWRDC AOAI responded: {reply}")
    print("\n✅ Cross-tenant validation complete.")


if __name__ == "__main__":
    validate()
```

---

## 4. Automation Artifacts — Summary

| Artifact | File | Tenant context | Runs when |
|---|---|---|---|
| Bicep: AOAI + model + RBAC | `nwrdc-aoai.bicep` | NWRDC | A1 (NWRDC admin) |
| CLI: App reg + federated cred | inline A2 | NWRDC | A2 (NWRDC admin) |
| CLI: UAMI creation | inline B1 | Agency | B1 (Agency admin) |
| CLI: UAMI → Hub assignment | inline B2 | Agency | B2 (Agency admin) |
| CLI: AI Search RBAC | inline B3 | Agency | B3 (Agency admin) |
| Python: WIF credential module | `cross_tenant_credential.py` | Agency compute | C2 (developer) |
| Python: Full agent | `agent.py` | Agency compute | C3 (developer) |
| Python: Validation script | `validate_cross_tenant.py` | Agency compute | C4 (developer) |

---

## 5. Security & Governance

### Least Privilege

| Who | Role | Scope | Why sufficient |
|---|---|---|---|
| NWRDC App SP | `Cognitive Services OpenAI User` | Specific AOAI resource | Has `DataActions` for chat completions, embeddings. Cannot view keys, cannot deploy models, cannot manage the account |
| Agency UAMI | `Search Index Data Reader` | Specific search index | Read-only retrieval; cannot modify index schema |
| Agency UAMI | `Search Service Contributor` | Search service | Required for index query execution |

### Role Reference

| Role | Role Definition ID | Inference via Entra | Deploy models | Manage account |
|---|---|---|---|---|
| Cognitive Services OpenAI **User** | `5e0bd9bd-7b93-4f28-af87-19fc36ad61bd` | ✅ | ❌ | ❌ |
| Cognitive Services OpenAI **Contributor** | `a001fd3d-188f-4b5d-821b-7da978bf7442` | ✅ | ✅ | ✅ |
| Cognitive Services **Contributor** | `25fbc0a9-bd7c-42a3-aa1a-3b75d497ee68` | ❌ (zero DataActions) | ✅ | ✅ |
| Cognitive Services **User** | `a97b65f3-24c7-4388-baec-2e87135dc908` | ✅ (all Cog Svcs) | ❌ | ❌ |

> **Warning**: `Cognitive Services Contributor` has **zero DataActions** and cannot call inference APIs with Entra ID. Using it instead of `Cognitive Services OpenAI User` is the most common misconfiguration in this pattern, resulting in `401` on all inference calls.

### Enforcing Keyless Access

`disableLocalAuth: true` on the NWRDC AOAI resource is **required**, not optional. It ensures that even if an API key is somehow obtained, it is rejected server-side. This eliminates credential-theft as an attack vector against the inference endpoint.

### Auditing

Enable diagnostics on the NWRDC AOAI resource:

```bash
# TENANT CONTEXT: NWRDC
az monitor diagnostic-settings create \
  --name "aoai-audit" \
  --resource "$AOAI_RESOURCE_ID" \
  --workspace "<log-analytics-workspace-id>" \
  --logs '[{"category": "Audit", "enabled": true}, {"category": "RequestResponse", "enabled": true}]'
```

KQL query to audit cross-tenant inference calls (NWRDC Log Analytics workspace):

```kql
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.COGNITIVESERVICES"
| where Category == "Audit"
| where operationName_s == "Chat Completions"
| extend CallerOID = tostring(parse_json(properties_s).callerObjectId)
| where CallerOID == "<nwrdc-app-sp-object-id>"
| project TimeGenerated, CallerOID, modelDeploymentName_s, promptTokens_d, completionTokens_d
| order by TimeGenerated desc
```

### RBAC Revocation

```bash
# TENANT CONTEXT: NWRDC

# Option 1: Remove the RBAC assignment (fastest — next inference call returns 401)
az role assignment delete \
  --assignee "$SP_OBJECT_ID" \
  --role "Cognitive Services OpenAI User" \
  --scope "$AOAI_RESOURCE_ID"

# Option 2: Delete the federated credential (blocks new token exchanges;
# existing tokens remain valid until expiry, typically 1 hour)
az ad app federated-credential delete \
  --id "$APP_ID" \
  --federated-credential-id agency-foundry-uami-fed

# Option 3: Disable the service principal entirely (immediate, hard block)
az ad sp update --id "$SP_OBJECT_ID" --set accountEnabled=false
```

### Common Misconfigurations

| Misconfiguration | Consequence | Correct approach |
|---|---|---|
| Using `Cognitive Services Contributor` instead of `Cognitive Services OpenAI User` | 401 on all inference calls (zero DataActions) | Use role ID `5e0bd9bd-7b93-4f28-af87-19fc36ad61bd` |
| Federated credential `subject` set to UAMI `clientId` instead of `principalId` | Token exchange fails with `AADSTS70021` | Use `az identity show --query principalId` |
| Federated credential `issuer` missing `/v2.0` suffix | Token exchange fails; issuer mismatch | Use `https://login.microsoftonline.com/{tenant-id}/v2.0` |
| RBAC scoped at subscription or resource group | Violates least privilege; all AOAI resources in scope exposed | Scope to specific AOAI resource ID only |
| `disableLocalAuth` not set | API key fallback remains active | Set `disableLocalAuth: true` at deploy time |
| `principalType: 'ServicePrincipal'` omitted in Bicep | Up to 30-minute RBAC propagation delay | Always include `principalType` |
| Using `DefaultAzureCredential` to acquire NWRDC token | Token issues for Agency tenant; rejected by NWRDC RBAC | Use explicit `ClientAssertionCredential` from `build_cross_tenant_credential()` |
| Using `"https://cognitiveservices.azure.com/.default"` scope for IMDS assertion | Returns Agency-tenant AOAI token, not a WIF assertion | Use `"api://AzureADTokenExchange"` for the IMDS step |

---

## 6. Troubleshooting

### External Model Not Appearing in Foundry Agent Builder UI

**Cause**: NWRDC AOAI not registered as a project connection, or connection type/auth is incompatible.

**Steps**:
1. Verify the connection exists: `az ml connection list --workspace-name <hub> --resource-group <rg>`
2. If missing, re-run `az ml connection create` from Phase B4
3. In Foundry portal → Project → Settings → Connected resources → confirm `nwrdc-aoai-connection` is listed
4. Refresh the agent builder UI (connections are cached)
5. For programmatic agents (Phase C code), the UI connection is not required — inference uses the `AzureOpenAI` client directly

---

### Token Exchange Fails: AADSTS70021

**Symptom**: `ClientAssertionCredential` raises `ClientAuthenticationError` with `AADSTS70021: No matching federated identity record found for presented assertion.`

**Diagnosis**:

```bash
# Verify the federated credential subject matches the UAMI's principalId
UAMI_PRINCIPAL_ID=$(az identity show \
  --name foundry-crosstenantmodel-mi \
  --resource-group agency-foundry-rg \
  --query principalId -o tsv)

# List the federated credential in NWRDC and compare
az ad app federated-credential list --id "$APP_ID"
# subject must == $UAMI_PRINCIPAL_ID
# issuer must == https://login.microsoftonline.com/<agency-tenant-id>/v2.0
# audiences must include api://AzureADTokenExchange
```

Decode the IMDS assertion claims on the compute:

```python
import jwt, os
from azure.identity import ManagedIdentityCredential

mi = ManagedIdentityCredential(client_id=os.environ["AGENCY_UAMI_CLIENT_ID"])
tok = mi.get_token("api://AzureADTokenExchange")
claims = jwt.decode(tok.token, options={"verify_signature": False})
print("iss:", claims.get("iss"))   # Must match federated credential issuer
print("sub:", claims.get("sub"))   # Must match federated credential subject
print("aud:", claims.get("aud"))   # Must include api://AzureADTokenExchange
```

---

### RBAC Propagation Delay (401 Immediately After Role Assignment)

**Explanation**: Entra ID RBAC assignments have eventual consistency. New assignments can take 5–30 minutes to propagate to all Azure Resource Manager caches.

**Mitigation**:
- Always include `principalType: 'ServicePrincipal'` in Bicep role assignments (avoids a secondary Entra lookup that adds latency)
- Wait at least 5 minutes after `az role assignment create` before running the validation script

**Monitor propagation**:

```bash
while true; do
  STATUS=$(az role assignment list \
    --assignee "$SP_OBJECT_ID" \
    --scope "$AOAI_RESOURCE_ID" \
    --query "[?roleDefinitionName=='Cognitive Services OpenAI User'].roleDefinitionName" \
    -o tsv)
  if [ "$STATUS" = "Cognitive Services OpenAI User" ]; then
    echo "RBAC propagated. Ready to validate."
    break
  fi
  echo "Waiting for RBAC propagation... (retrying in 30s)"
  sleep 30
done
```

---

### B2B Guest Cannot Call AOAI / B2B Invitation Issues for Machine Identity

**Root cause**: B2B guest collaboration is designed for human identities only. Managed identities:
- Cannot receive a B2B invitation (invitations require an email address)
- Cannot redeem a B2B invitation (redemption requires interactive browser-based consent)
- Cannot be represented as guest user objects across tenants

**Resolution**: Machine-to-machine cross-tenant access requires Workload Identity Federation (implemented in this plan). B2B is the correct pattern only for human developer testing:

```bash
# For a human developer to test NWRDC AOAI directly:
# 1. NWRDC admin invites the developer's user account as a B2B guest
# 2. NWRDC admin assigns Cognitive Services OpenAI User to the guest user object
# 3. Developer logs in to the NWRDC tenant locally:
az login --tenant <nwrdc-tenant-id> --allow-no-subscriptions
# 4. DefaultAzureCredential in dev code then uses the NWRDC session token
```

---

### Token Audience / Tenant Mismatch (401 with "Principal does not have access")

**Diagnostic — decode the token being sent**:

```python
import jwt

claims = jwt.decode(bearer_token, options={"verify_signature": False})
print("tid (token tenant):", claims["tid"])   # Must be NWRDC tenant ID
print("aud (audience):",     claims["aud"])   # Must be https://cognitiveservices.azure.com
print("oid (principal):",    claims["oid"])   # Must match NWRDC SP object ID
```

**Symptom → cause mapping**:

| Observed token claim | Likely cause | Fix |
|---|---|---|
| `tid` = Agency tenant ID | `DefaultAzureCredential` used instead of `ClientAssertionCredential` for NWRDC call | Always use `build_cross_tenant_credential()` for NWRDC calls |
| `oid` = UAMI object ID, `tid` = Agency | WIF exchange not triggered; IMDS JWT used directly | Ensure `ClientAssertionCredential` wraps the IMDS assertion |
| `aud` ≠ `cognitiveservices.azure.com` | Wrong scope in `get_bearer_token_provider` | Use `"https://cognitiveservices.azure.com/.default"` |
| `AADSTS700016` during exchange | Wrong `client_id` posted to token endpoint | Verify `NWRDC_APP_CLIENT_ID` environment variable |

---

## Appendix A — Adding a Custom REST API Tool

This appendix covers the scenario where the Agency agent needs to **call a custom REST API as an action tool** — for example, submitting a request to a state line-of-business system, retrieving a citizen record from a custom backend, or triggering an approval workflow. This is separate from and complementary to the cross-tenant model pattern in the main plan.

### Concept: Model vs. Tool

| | Cross-tenant AOAI (main plan) | Custom REST API tool (this appendix) |
|---|---|---|
| What it is | The LLM that reasons and generates responses | An external system the LLM can call during reasoning |
| Configured as | `model=` parameter on agent create | `OpenApiTool` registered on the agent |
| Auth | Workload Identity Federation (WIF) | API key, OAuth, or Managed Identity (see below) |
| OpenAPI spec needed? | ❌ No | ✅ Yes — OpenAPI 3.0 |
| Foundry UI feature | "Connected resources" | "Create an OpenAPI tool" |

The agent uses the NWRDC model to *think*, and the OpenAPI tool to *act*. Both can be present on the same agent.

---

### Step 1 — Write the OpenAPI 3.0 Spec for Your API

The spec must describe every endpoint the agent is allowed to call. The LLM reads the `description` fields to decide when and how to invoke each operation — write them as instructions, not documentation.

**File**: `agency-backend-api.yaml`

```yaml
openapi: "3.0.4"
info:
  title: Agency Backend API
  description: >
    Internal Agency API for citizen record lookup and workflow submission.
    Use this tool to retrieve citizen data by ID or to submit a workflow request.
    Do not call this API unless the user has explicitly requested an action that
    requires citizen data or workflow submission.
  version: "1.0.0"
servers:
  - url: https://api.agency.gov.au/v1

paths:
  /citizens/{citizenId}:
    get:
      operationId: getCitizenRecord
      summary: Retrieve a citizen record by ID
      description: >
        Returns name, address, and service eligibility flags for a given citizen ID.
        Only call this when the user provides a citizen ID and asks for their record.
      parameters:
        - name: citizenId
          in: path
          required: true
          schema:
            type: string
          description: The citizen's unique identifier (format: CIT-XXXXXXXX)
      responses:
        "200":
          description: Citizen record
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/CitizenRecord"
        "404":
          description: Citizen not found

  /workflows:
    post:
      operationId: submitWorkflow
      summary: Submit a workflow request
      description: >
        Submits a new workflow request on behalf of the user.
        Only call this after the user has confirmed they want to submit.
        Do not submit without explicit user confirmation.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/WorkflowRequest"
      responses:
        "201":
          description: Workflow created
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/WorkflowResponse"

components:
  schemas:
    CitizenRecord:
      type: object
      properties:
        citizenId:
          type: string
        fullName:
          type: string
        address:
          type: string
        eligibilityFlags:
          type: array
          items:
            type: string

    WorkflowRequest:
      type: object
      required: [citizenId, workflowType, requestedBy]
      properties:
        citizenId:
          type: string
        workflowType:
          type: string
          enum: [DataAmendment, ServiceEnrolment, AppealRequest]
        requestedBy:
          type: string
        notes:
          type: string

    WorkflowResponse:
      type: object
      properties:
        workflowId:
          type: string
        status:
          type: string
        createdAt:
          type: string
          format: date-time

  securitySchemes:
    ApiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key

security:
  - ApiKeyAuth: []
```

> **Security note**: The `securitySchemes` block is required in the spec for Foundry to know how to authenticate. Store the actual key in a Foundry connection (see Step 2), never in the spec file itself.

---

### Step 2 — Create a Foundry Connection for the API Credential

The API key (or other credential) is stored as a Foundry connection, not in code or the spec file.

**Option A — Portal**:
1. AI Foundry portal → your project → **Management** → **Connected resources** → **+ New connection**
2. Select **Custom** (or **Generic API**)
3. Enter the base URL: `https://api.agency.gov.au/v1`
4. Add a header credential: key = `X-API-Key`, value = `<your-api-key>`
5. Name it `agency-backend-api` and save

**Option B — REST API**:

```bash
# TENANT CONTEXT: AGENCY
AGENCY_SUB_ID=$(az account show --query id -o tsv)
ACCESS_TOKEN=$(az account get-access-token \
  --resource https://management.azure.com \
  --query accessToken -o tsv)

curl -s -X PUT \
  "https://management.azure.com/subscriptions/${AGENCY_SUB_ID}/resourceGroups/${FOUNDRY_RG}/providers/Microsoft.CognitiveServices/accounts/${FOUNDRY_RESOURCE_NAME}/projects/${FOUNDRY_PROJECT_NAME}/connections/agency-backend-api?api-version=2025-04-01-preview" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "properties": {
      "category": "CustomKeys",
      "target": "https://api.agency.gov.au/v1",
      "authType": "CustomKeys",
      "credentials": {
        "keys": {
          "X-API-Key": "<your-api-key>"
        }
      }
    }
  }'
```

---

### Step 3 — Register the OpenAPI Tool on the Agent (SDK)

Add the `OpenApiTool` alongside the existing `AzureAISearchTool` when creating the agent.

**File**: `agent_with_openapi.py` — extends `agent.py` from Phase C3

```python
import json
import os
from pathlib import Path
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import (
    AzureAISearchTool,
    AzureAISearchQueryType,
    OpenApiTool,
    OpenApiConnectionAuthDetails,
    OpenApiConnectionSecurityScheme,
    ConnectionType,
)
from azure.identity import DefaultAzureCredential
from cross_tenant_credential import build_cross_tenant_credential
from openai import AzureOpenAI
from azure.identity import get_bearer_token_provider

# ── Clients (same as agent.py) ────────────────────────────────────────────────
nwrdc_credential = build_cross_tenant_credential(
    uami_client_id=os.environ["AGENCY_UAMI_CLIENT_ID"],
    nwrdc_tenant_id=os.environ["NWRDC_TENANT_ID"],
    nwrdc_app_client_id=os.environ["NWRDC_APP_CLIENT_ID"],
)

agency_foundry_client = AIProjectClient(
    endpoint=os.environ["AGENCY_FOUNDRY_ENDPOINT"],
    credential=DefaultAzureCredential(
        managed_identity_client_id=os.environ["AGENCY_UAMI_CLIENT_ID"]
    ),
)

# ── Resolve connections ───────────────────────────────────────────────────────
search_connection = agency_foundry_client.connections.get_default(
    connection_type=ConnectionType.AZURE_AI_SEARCH
)

# Retrieve the backend API connection by name
backend_api_connection = agency_foundry_client.connections.get(
    connection_name="agency-backend-api"
)


def create_agent_with_openapi_tool(agent_name: str = "agency-full-agent"):
    """
    Creates an agent with both:
      - AzureAISearchTool  → knowledge retrieval (same-tenant)
      - OpenApiTool        → custom REST API actions (same-tenant backend)
      - NWRDC gpt-4o       → model inference (cross-tenant via WIF)
    """
    # Load the OpenAPI spec from file
    spec_text = Path("agency-backend-api.yaml").read_text()

    # Configure the OpenAPI tool to authenticate using the Foundry connection
    # The connection stores the X-API-Key header value securely.
    openapi_tool = OpenApiTool(
        name="agency_backend",
        spec=spec_text,
        description="Agency backend API for citizen record lookup and workflow submission.",
        auth=OpenApiConnectionAuthDetails(
            security_scheme=OpenApiConnectionSecurityScheme(
                connection_id=backend_api_connection.id
            )
        ),
    )

    search_tool = AzureAISearchTool(
        connection_id=search_connection.id,
        index_name=os.environ["AGENCY_SEARCH_INDEX"],
        query_type=AzureAISearchQueryType.VECTOR_SEMANTIC_HYBRID,
    )

    # Combine tool definitions and resources from both tools
    all_tool_definitions = (
        search_tool.definitions
        + openapi_tool.definitions
    )
    # AzureAISearchTool uses tool_resources; OpenApiTool does not
    tool_resources = search_tool.resources

    agent = agency_foundry_client.agents.create_agent(
        model=os.environ["NWRDC_MODEL_DEPLOYMENT"],
        name=agent_name,
        instructions=(
            "You are a helpful assistant for Agency staff. "
            "Use the knowledge base search tool to answer policy and procedure questions. "
            "Use the agency_backend tool to look up citizen records or submit workflow requests "
            "— but only when the user explicitly requests it and only after confirming any "
            "write actions (submitWorkflow) with the user before proceeding."
        ),
        tools=all_tool_definitions,
        tool_resources=tool_resources,
    )
    print(f"Created agent with OpenAPI tool: {agent.id}")
    return agent
```

---

### Step 4 — Register via the Foundry Portal UI (alternative to SDK)

If the Agency team prefers the portal's guided experience:

1. AI Foundry portal → your project → **Agents** → open your agent → **Tools** → **+ Add tool**
2. Select **OpenAPI 3.0 compatible API**
3. Upload `agency-backend-api.yaml` (or paste the spec inline)
4. Under **Authentication**, select **Connection** → choose `agency-backend-api`
5. Save the agent

The portal stores exactly the same configuration as the SDK approach above.

---

### Auth Options for the Custom API

The `securitySchemes` in the OpenAPI spec and the Foundry connection type must match.

| Auth type | OpenAPI `securitySchemes` type | Foundry connection `authType` | When to use |
|---|---|---|---|
| API Key (header or query) | `apiKey` | `CustomKeys` | Simple internal APIs, API Management-fronted APIs |
| OAuth 2.0 client credentials | `oauth2` (flows: clientCredentials) | `OAuth2` | APIs secured by Entra ID app registration |
| Managed Identity (Entra ID) | `http` (bearer) | `ManagedIdentity` | APIs hosted in Azure that accept Entra tokens |
| Anonymous | none | `None` | Internal network-only APIs with no auth (not recommended) |

For Agency backend APIs secured by Entra ID (recommended), use `ManagedIdentity` auth so the UAMI's token is forwarded — no API key stored anywhere:

```yaml
# In the OpenAPI spec securitySchemes block:
securitySchemes:
  EntraID:
    type: http
    scheme: bearer
    bearerFormat: JWT
```

```bash
# Foundry connection for Managed Identity auth — no credential stored
curl -s -X PUT \
  "https://management.azure.com/.../connections/agency-backend-api?api-version=2025-04-01-preview" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "properties": {
      "category": "CustomKeys",
      "target": "https://api.agency.gov.au/v1",
      "authType": "ManagedIdentity"
    }
  }'
```

The UAMI must hold the appropriate RBAC role (or be in the allowed identities list) on the backend API's Entra app registration.

---

### Security Considerations for OpenAPI Tools

| Consideration | Guidance |
|---|---|
| Scope the spec tightly | Only expose endpoints the agent actually needs. Do not include admin or delete endpoints unless required. |
| Write defensive `description` fields | The LLM uses descriptions to decide when to call an operation. Include guardrails: "Only call after user confirmation", "Do not call unless user provides X". |
| Prefer read-only operations by default | Add write/mutating operations only if the use case requires them, and always add a human confirmation step in the instructions. |
| Store credentials in Foundry connections | Never embed API keys in the spec file, in code, or in agent instructions. |
| Validate on the API side | The backend API must independently validate the caller identity. The agent is not a trust boundary — treat it as an untrusted client. |
| Log all tool invocations | Enable Foundry agent telemetry (Application Insights) so every `getCitizenRecord` and `submitWorkflow` call is auditable with the thread ID and run ID. |
