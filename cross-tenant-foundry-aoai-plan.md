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
│  │  Azure AI Foundry Project (existing)        │      │
│  │  ├── System/User-Assigned MI (UAMI)         │      │
│  │  ├── AI Agents (azure-ai-projects SDK)      │      │
│  │  ├── Azure AI Search connection (same-      │      │
│  │  │   tenant, standard Entra ID RBAC)        │      │
│  │  └── Threads, Evaluations, Storage         │      │
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
| Attach UAMI to Foundry project compute | | ✅ | | Required |
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

**Prerequisites**: Logged into Azure CLI as Agency tenant contributor. Agency AI Foundry project is already provisioned.

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

#### B2. Assign the UAMI to the Foundry Hub

```bash
# TENANT CONTEXT: AGENCY

FOUNDRY_HUB_NAME="agency-ai-hub"
FOUNDRY_HUB_RG="agency-foundry-rg"

# Assign UAMI to the Foundry Hub
az ml workspace update \
  --name "$FOUNDRY_HUB_NAME" \
  --resource-group "$FOUNDRY_HUB_RG" \
  --assign-identity "$UAMI_RESOURCE_ID"

# Verify: AI Foundry Hub → Identity → User assigned → shows foundry-crosstenantmodel-mi
```

For Container Apps or other agent compute:

```bash
az containerapp identity assign \
  --name <containerapp-name> \
  --resource-group "$AGENCY_RG" \
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

> Registers the NWRDC AOAI endpoint in the Foundry portal UI. Auth for programmatic calls is handled by developer code (Phase C), not through this connection's Entra ID route, due to the cross-tenant limitation of Foundry's native connection auth.

```bash
# TENANT CONTEXT: AGENCY

az ml connection create \
  --name "nwrdc-aoai-connection" \
  --resource-group "$FOUNDRY_HUB_RG" \
  --workspace-name "$FOUNDRY_HUB_NAME" \
  --file nwrdc-connection.yaml
```

**File**: `nwrdc-connection.yaml`

```yaml
# Connection metadata only — auth is handled programmatically in agent code.
# disableLocalAuth: true on the NWRDC AOAI resource ensures this placeholder
# can never be used for a real inference call even if extracted.
name: nwrdc-aoai-connection
type: azure_open_ai
endpoint: https://nwrdc-aoai-prod.openai.azure.com/
api_type: azure
api_version: "2024-10-21"
auth_type: custom_keys
credentials:
  key: "PLACEHOLDER_REPLACED_IN_CODE"
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

# Agency Foundry project endpoint
AGENCY_FOUNDRY_ENDPOINT=https://<hub-name>.services.ai.azure.com/api/projects/<project-name>

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
