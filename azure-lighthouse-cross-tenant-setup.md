# Azure Lighthouse Cross-Tenant Access Configuration

## NWRDC (Foundry Tenant) ↔ Agency Tenant

**Goal:** Grant NWRDC member users access to Agency tenant resources (Azure AI Search, Storage, etc.)
without directory switching, using Azure Lighthouse + RBAC.

---

## Fill-In Reference Variables

Replace all placeholders throughout before running any commands.

| Placeholder | Description |
|---|---|
| `<NWRDC_TENANT_ID>` | NWRDC Directory (tenant) ID |
| `<NWRDC_SUB_ID>` | NWRDC subscription ID (where AI Foundry lives) |
| `<NWRDC_FOUNDRY_RG>` | Resource group containing Azure AI Foundry |
| `<NWRDC_DOMAIN>` | NWRDC default domain (e.g. `nwrdc.onmicrosoft.com`) |
| `<AGENCY_TENANT_ID>` | Agency Directory ID |
| `<AGENCY_SUB_ID>` | Agency subscription to delegate |
| `<AGENCY_RG>` | (RG-scope only) Agency resource group to delegate |
| `<PRINCIPAL_ID>` | Object ID of NWRDC user or security group |

---

## Part A — NWRDC / Foundry Tenant Admin

### A1. Create Member User Accounts

```bash
# Sign in to the NWRDC tenant
az login --tenant <NWRDC_TENANT_ID>

# Create a member user (NOT a B2B guest invite)
az ad user create \
  --display-name "Agency User Name" \
  --user-principal-name "firstname.lastname@<NWRDC_DOMAIN>" \
  --password "ChangeMe@12345!" \
  --force-change-password-next-sign-in true \
  --mail-nickname "firstname.lastname"

# Capture the new user's object ID (needed for the Lighthouse template)
az ad user show \
  --id "firstname.lastname@<NWRDC_DOMAIN>" \
  --query id \
  --output tsv
```

**PowerShell (Microsoft Graph) equivalent:**

```powershell
# Install-Module Microsoft.Graph -Scope CurrentUser  # run once if needed
Connect-MgGraph -TenantId "<NWRDC_TENANT_ID>" -Scopes "User.ReadWrite.All"

$passwordProfile = @{
    Password                      = "ChangeMe@12345!"
    ForceChangePasswordNextSignIn = $true
}

$user = New-MgUser `
    -DisplayName "Agency User Name" `
    -UserPrincipalName "firstname.lastname@<NWRDC_DOMAIN>" `
    -PasswordProfile $passwordProfile `
    -AccountEnabled:$true `
    -MailNickname "firstname.lastname"

Write-Output "User Object ID: $($user.Id)"
```

> Repeat for each agency user. Enforce MFA enrollment on first sign-in via Conditional Access in the NWRDC tenant.

---

### A2. (Recommended) Create a Security Group for Delegation

Assigning a group rather than individual users makes adding/removing access easier without redeploying the Lighthouse template.

```bash
# Create the security group in the NWRDC tenant
az ad group create \
  --display-name "NWRDC-Foundry-AgencyAccess" \
  --mail-nickname "nwrdc-foundry-agency-access"

# Get the group's object ID — use this as <PRINCIPAL_ID> in the Lighthouse template
az ad group show \
  --group "NWRDC-Foundry-AgencyAccess" \
  --query id \
  --output tsv

# Add a user to the group
az ad group member add \
  --group "NWRDC-Foundry-AgencyAccess" \
  --member-id "<USER-OBJECT-ID>"
```

---

### A3. Grant Azure AI Foundry Access (within NWRDC Tenant)

```bash
# Assign Azure AI Developer role at the Foundry resource group
az role assignment create \
  --assignee "<PRINCIPAL_ID>" \
  --role "Azure AI Developer" \
  --scope "/subscriptions/<NWRDC_SUB_ID>/resourceGroups/<NWRDC_FOUNDRY_RG>"

# --- OR --- assign at subscription level
az role assignment create \
  --assignee "<PRINCIPAL_ID>" \
  --role "Azure AI Developer" \
  --scope "/subscriptions/<NWRDC_SUB_ID>"
```

---

### A4. Collect IDs Needed for the Lighthouse Template

```bash
# NWRDC tenant ID (run while signed in to NWRDC tenant)
az account show --query tenantId --output tsv

# User object ID
az ad user show --id "firstname.lastname@<NWRDC_DOMAIN>" --query id --output tsv

# Group object ID (if using a group — preferred)
az ad group show --group "NWRDC-Foundry-AgencyAccess" --query id --output tsv
```

---

### A5. Lighthouse Template Files (send to Agency Admin)

Create the files below and send them to the Agency tenant admin.

---

#### Option A — Subscription-Scoped Delegation (recommended)

Delegates access to the entire Agency subscription.

**`lighthouse-offer.json`** (ARM):

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2018-05-01/subscriptionDeploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "managedByTenantId": {
      "type": "string",
      "metadata": { "description": "Tenant ID of the NWRDC (managing) tenant." }
    },
    "principalId": {
      "type": "string",
      "metadata": { "description": "Object ID of the NWRDC user or security group." }
    },
    "principalDisplayName": {
      "type": "string",
      "defaultValue": "NWRDC Foundry Operators",
      "metadata": { "description": "Label shown in the Agency portal under 'Service providers'." }
    },
    "offerName": {
      "type": "string",
      "defaultValue": "NWRDC Foundry Access",
      "metadata": { "description": "Friendly name for this delegation offer." }
    }
  },
  "variables": {
    "contributorRoleId": "b24988ac-6180-42a0-ab88-20f7382dd24c",
    "definitionName": "[guid('lh-def', parameters('managedByTenantId'), parameters('principalId'))]",
    "assignmentName": "[guid('lh-asgn', parameters('managedByTenantId'), parameters('principalId'))]"
  },
  "resources": [
    {
      "type": "Microsoft.ManagedServices/registrationDefinitions",
      "apiVersion": "2022-10-01",
      "name": "[variables('definitionName')]",
      "properties": {
        "registrationDefinitionName": "[parameters('offerName')]",
        "description": "Delegates Contributor access to NWRDC Foundry operators for cross-tenant AI workloads.",
        "managedByTenantId": "[parameters('managedByTenantId')]",
        "authorizations": [
          {
            "principalId": "[parameters('principalId')]",
            "principalIdDisplayName": "[parameters('principalDisplayName')]",
            "roleDefinitionId": "[variables('contributorRoleId')]"
          }
        ]
      }
    },
    {
      "type": "Microsoft.ManagedServices/registrationAssignments",
      "apiVersion": "2022-10-01",
      "name": "[variables('assignmentName')]",
      "dependsOn": [
        "[subscriptionResourceId('Microsoft.ManagedServices/registrationDefinitions', variables('definitionName'))]"
      ],
      "properties": {
        "registrationDefinitionId": "[subscriptionResourceId('Microsoft.ManagedServices/registrationDefinitions', variables('definitionName'))]"
      }
    }
  ]
}
```

**`lighthouse-offer.bicep`** (Bicep equivalent):

```bicep
// lighthouse-offer.bicep
// Agency admin deploys with:
//   az deployment sub create --location eastus --template-file lighthouse-offer.bicep \
//     --parameters managedByTenantId=<NWRDC_TENANT_ID> principalId=<PRINCIPAL_ID>
targetScope = 'subscription'

@description('Tenant ID of the NWRDC (managing) tenant.')
param managedByTenantId string

@description('Object ID of the NWRDC user or security group.')
param principalId string

@description('Label shown in the Agency portal under Service providers.')
param principalDisplayName string = 'NWRDC Foundry Operators'

@description('Friendly name for this delegation offer.')
param offerName string = 'NWRDC Foundry Access'

var contributorRoleId = 'b24988ac-6180-42a0-ab88-20f7382dd24c'
var definitionName = guid('lh-def', managedByTenantId, principalId)
var assignmentName = guid('lh-asgn', managedByTenantId, principalId)

resource registrationDefinition 'Microsoft.ManagedServices/registrationDefinitions@2022-10-01' = {
  name: definitionName
  properties: {
    registrationDefinitionName: offerName
    description: 'Delegates Contributor access to NWRDC Foundry operators for cross-tenant AI workloads.'
    managedByTenantId: managedByTenantId
    authorizations: [
      {
        principalId: principalId
        principalIdDisplayName: principalDisplayName
        roleDefinitionId: contributorRoleId
      }
    ]
  }
}

resource registrationAssignment 'Microsoft.ManagedServices/registrationAssignments@2022-10-01' = {
  name: assignmentName
  properties: {
    registrationDefinitionId: registrationDefinition.id
  }
}
```

---

#### Option B — Resource Group-Scoped Delegation (more restrictive)

Delegates access to a single Agency resource group. Requires two sequential deployments because the
`registrationDefinition` resource must be registered at subscription scope while the assignment is
scoped to the resource group.

**Step 1 — `lighthouse-definition.bicep`** (subscription scope, creates the definition):

```bicep
// lighthouse-definition.bicep
// Agency admin deploys with:
//   az deployment sub create --location eastus --template-file lighthouse-definition.bicep \
//     --parameters managedByTenantId=<NWRDC_TENANT_ID> principalId=<PRINCIPAL_ID>
targetScope = 'subscription'

@description('Tenant ID of the NWRDC (managing) tenant.')
param managedByTenantId string

@description('Object ID of the NWRDC user or security group.')
param principalId string

param principalDisplayName string = 'NWRDC Foundry Operators'
param offerName string = 'NWRDC Foundry Access (RG)'

var contributorRoleId = 'b24988ac-6180-42a0-ab88-20f7382dd24c'
var definitionName = guid('lh-def-rg', managedByTenantId, principalId)

resource registrationDefinition 'Microsoft.ManagedServices/registrationDefinitions@2022-10-01' = {
  name: definitionName
  properties: {
    registrationDefinitionName: offerName
    description: 'Delegates Contributor access to NWRDC Foundry operators (resource group scope).'
    managedByTenantId: managedByTenantId
    authorizations: [
      {
        principalId: principalId
        principalIdDisplayName: principalDisplayName
        roleDefinitionId: contributorRoleId
      }
    ]
  }
}

output definitionId string = registrationDefinition.id
```

**Step 2 — `lighthouse-assignment-rg.bicep`** (resource group scope, creates the assignment):

```bicep
// lighthouse-assignment-rg.bicep
// Agency admin deploys with:
//   az deployment group create --resource-group <AGENCY_RG> \
//     --template-file lighthouse-assignment-rg.bicep \
//     --parameters definitionId=<OUTPUT_DEFINITION_ID_FROM_STEP1>

@description('Full resource ID of the registrationDefinition output from Step 1.')
param definitionId string

var assignmentName = guid('lh-asgn-rg', definitionId)

resource registrationAssignment 'Microsoft.ManagedServices/registrationAssignments@2022-10-01' = {
  name: assignmentName
  properties: {
    registrationDefinitionId: definitionId
  }
}
```

---

## Part B — Agency Tenant Admin

### B1. Deploy — Subscription Scope (Option A)

```bash
# Sign in to the Agency tenant
az login --tenant <AGENCY_TENANT_ID>
az account set --subscription <AGENCY_SUB_ID>

# Deploy the ARM JSON template
az deployment sub create \
  --location eastus \
  --template-file lighthouse-offer.json \
  --parameters managedByTenantId=<NWRDC_TENANT_ID> \
               principalId=<PRINCIPAL_ID> \
               principalDisplayName="NWRDC Foundry Operators" \
               offerName="NWRDC Foundry Access"

# --- OR --- deploy the Bicep template
az deployment sub create \
  --location eastus \
  --template-file lighthouse-offer.bicep \
  --parameters managedByTenantId=<NWRDC_TENANT_ID> \
               principalId=<PRINCIPAL_ID>
```

### B2. Deploy — Resource Group Scope (Option B)

```bash
az login --tenant <AGENCY_TENANT_ID>
az account set --subscription <AGENCY_SUB_ID>

# Step 1: Register the definition at subscription scope; capture the output ID
DEFINITION_ID=$(az deployment sub create \
  --location eastus \
  --template-file lighthouse-definition.bicep \
  --parameters managedByTenantId=<NWRDC_TENANT_ID> principalId=<PRINCIPAL_ID> \
  --query "properties.outputs.definitionId.value" \
  --output tsv)

echo "Definition ID: $DEFINITION_ID"

# Step 2: Create the assignment scoped to the target resource group
az deployment group create \
  --resource-group <AGENCY_RG> \
  --template-file lighthouse-assignment-rg.bicep \
  --parameters definitionId="$DEFINITION_ID"
```

---

## Terraform Alternative

Run by the Agency tenant admin. Implements subscription-scope delegation (equivalent to Option A above).

**`providers.tf`:**

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features        {}
  tenant_id       = var.agency_tenant_id
  subscription_id = var.agency_subscription_id
}
```

**`variables.tf`:**

```hcl
variable "nwrdc_tenant_id" {
  description = "Tenant ID of the NWRDC managing tenant."
  type        = string
}

variable "nwrdc_principal_id" {
  description = "Object ID of NWRDC user or security group."
  type        = string
}

variable "nwrdc_principal_display_name" {
  description = "Display name for the managing entity."
  type        = string
  default     = "NWRDC Foundry Operators"
}

variable "agency_tenant_id" {
  description = "Agency tenant ID."
  type        = string
}

variable "agency_subscription_id" {
  description = "Agency subscription to delegate."
  type        = string
}
```

**`main.tf`:**

```hcl
locals {
  contributor_role_id = "b24988ac-6180-42a0-ab88-20f7382dd24c"
}

resource "azurerm_lighthouse_definition" "nwrdc_access" {
  name               = "NWRDC Foundry Access"
  description        = "Delegates Contributor access to NWRDC Foundry operators for cross-tenant AI workloads."
  managing_tenant_id = var.nwrdc_tenant_id
  scope              = "/subscriptions/${var.agency_subscription_id}"

  authorization {
    principal_id           = var.nwrdc_principal_id
    principal_display_name = var.nwrdc_principal_display_name
    role_definition_id     = local.contributor_role_id
  }
}

resource "azurerm_lighthouse_assignment" "nwrdc_access" {
  scope                    = "/subscriptions/${var.agency_subscription_id}"
  lighthouse_definition_id = azurerm_lighthouse_definition.nwrdc_access.id
}
```

**`terraform.tfvars`** (do not commit to source control):

```hcl
nwrdc_tenant_id              = "<NWRDC_TENANT_ID>"
nwrdc_principal_id           = "<PRINCIPAL_ID>"
nwrdc_principal_display_name = "NWRDC Foundry Operators"
agency_tenant_id             = "<AGENCY_TENANT_ID>"
agency_subscription_id       = "<AGENCY_SUB_ID>"
```

```bash
# Deploy
terraform init
terraform plan -out=lighthouse.plan
terraform apply lighthouse.plan
```

---

## Part C — Verification Commands

### Verify from NWRDC Tenant (as the delegated user)

```bash
# Sign in as the NWRDC member user
az login --tenant <NWRDC_TENANT_ID>

# List all accessible subscriptions — both NWRDC and Agency should appear
az account list --output table

# Switch context to the Agency subscription (no tenant switch required)
az account set --subscription <AGENCY_SUB_ID>

# Confirm resource visibility in the Agency subscription
az resource list --subscription <AGENCY_SUB_ID> --output table

# List AI Search services in the Agency resource group
az search service list \
  --resource-group <AGENCY_RG> \
  --subscription <AGENCY_SUB_ID> \
  --output table

# List storage accounts in the Agency subscription
az storage account list \
  --subscription <AGENCY_SUB_ID> \
  --output table
```

### Verify Lighthouse Delegation (from Agency tenant)

```bash
az login --tenant <AGENCY_TENANT_ID>

# Confirm the registration assignment exists
az managedservices assignment list --output table

# Confirm the definition
az managedservices definition list --output table
```

### Verify RBAC Assignments on the Delegated Scope

```bash
az login --tenant <AGENCY_TENANT_ID>

# List all role assignments (delegated entries will show the NWRDC tenant's principal)
az role assignment list \
  --subscription <AGENCY_SUB_ID> \
  --include-classic-administrators false \
  --output table
```

---

## Key Constraints and Notes

| Constraint | Detail |
|---|---|
| **No B2B guests** | Only member (organizational) accounts in the NWRDC tenant work. B2B guest invitations are not supported for Azure Lighthouse delegation. |
| **No AI-specific Lighthouse roles** | The Lighthouse role picker does not expose Azure AI–specific built-in roles. `Contributor` (`b24988ac-6180-42a0-ab88-20f7382dd24c`) is used as a broad alternative. |
| **PIM not required** | Access type is `Permanent` (not `Eligible`) to avoid PIM licensing requirements. This can be changed to Eligible later if PIM licenses are available. |
| **Identity context** | Cross-tenant access operates under the user's own NWRDC identity — it is not automatic or service-to-service. Azure AI Foundry must be configured to reference specific agency resources explicitly. |
| **Propagation delay** | After the Agency deploys the template, allow up to 15 minutes. Sign out and back in to the NWRDC tenant if subscriptions do not appear immediately. |
| **Blast radius** | Contributor at subscription scope grants full create/read/update/delete on all resources in that subscription. Use Option B (resource group scope) for a narrower delegation surface. |
| **Revoking access** | The Agency admin can remove the delegation at any time by deleting the `registrationAssignment` resource or re-running the Lighthouse template with the assignment removed. The NWRDC admin cannot unilaterally revoke the agency's own subscription; only the agency can. |
