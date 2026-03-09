# Cherwell Knowledge Mining Pipeline

> **FLHack Project** | Built with Microsoft AI Foundry & VS Code

## Introduction

Many IT organizations accumulate years of resolved support tickets in tools like Cherwell, but that institutional knowledge is rarely searchable or reusable. Agents spend time re-solving the same problems, and new staff have no structured way to learn from past resolutions.

This project builds an **automated knowledge mining pipeline** that extracts value from historical Cherwell ITSM data and transforms it into a living, AI-curated knowledge base in SharePoint — powered by Microsoft AI Foundry.

The system:
- **Pulls** resolved tickets, work notes, and attachments from Cherwell via REST API
- **Normalizes** raw ticket data into a structured canonical model using Azure Functions
- **Processes** each ticket through a 5-agent AI pipeline on Azure AI Foundry — extracting, classifying, summarizing, validating, and publishing the knowledge
- **Publishes** polished, PII-scrubbed knowledge base articles directly to a SharePoint list
- **Indexes** all content in Azure AI Search for fast semantic retrieval
- **Monitors** the entire pipeline with Azure Monitor and Application Insights

The result is a continuously updated knowledge base that captures institutional IT expertise — automatically, at scale.

---

## System Architecture

![API-Based Ingestion and Knowledge Mining for Cherwell to SharePoint](docs/architecture.png)

The diagram above shows the full data flow: from Cherwell on-premises through the Azure Function ingestion layer, into the AI Foundry 5-agent processing pipeline, and finally published to the SharePoint Knowledge Base — with Azure AI Search and monitoring across the bottom tier.

---

## Architecture Overview

```
On-Prem Cherwell  →  Azure Function (Ingest)  →  Azure Blob Storage
                                                        ↓
                                              Azure AI Foundry Pipeline
                                    ┌──────────────────────────────────────┐
                                    │  Extraction → Classification          │
                                    │  → Summarization → Validation         │
                                    │  → Publishing                         │
                                    └──────────────────────────────────────┘
                                                        ↓
                                         SharePoint Knowledge Base
                                      (IT KB List + AI Search Index)

Monitoring: Azure Monitor + Application Insights
```

---

## Build Phases

### Phase 1 — Prerequisites & Infrastructure

> Provision all Azure resources before writing any agent code.

| Step | Action |
|------|--------|
| 1.1 | Install VS Code extensions: AI Toolkit, Azure Functions, Azure Resources |
| 1.2 | Create an **Azure AI Foundry project** (Standard setup — bring your own storage + search) |
| 1.3 | Deploy **Azure Blob Storage** with containers: `/curated/tickets`, `/worklogs`, `/attachments` |
| 1.4 | Deploy **Azure AI Search**, configure indexer over Blob Storage |
| 1.5 | Deploy **Azure Function App** (Python, HTTP + Timer triggers) |
| 1.6 | Deploy **Application Insights**, link to Foundry project |
| 1.7 | Assign RBAC: Function → Blob (`Storage Blob Data Contributor`), Agent → Search (`Search Index Data Reader`) |

**What you'll need from your Cherwell admin:**
- Base URL (e.g., `https://cherwell.contoso.com/CherwellAPI`)
- API Key
- Service account credentials
- Business Object IDs for Incidents / Journals

---

### Phase 2 — Data Ingestion (Azure Function)

> Pull Cherwell tickets, normalize them, and land in Blob Storage.

**VS Code workflow:** Scaffold with `func init`, implement two modules:

- `data_extraction.py` — calls Cherwell REST/OData API, pulls Incidents, Work Notes, Attachments
- `data_normalization.py` — transforms to canonical JSON model

**Canonical ticket model:**
```json
{
  "ticketId": "INC12345",
  "description": "...",
  "resolution": "...",
  "workNotes": ["..."],
  "attachments": ["blob://curated/attachments/INC12345_screenshot.png"],
  "service": null,
  "issueType": null,
  "createdDate": "2026-03-09T00:00:00Z"
}
```

- Secrets (API Key, connection strings) stored in **Azure Key Vault**
- **Timer Trigger** for scheduled pulls; **HTTP Trigger** for on-demand

---

### Phase 3 — AI Foundry Agent Pipeline (5 Agents)

> Build the multi-agent pipeline using Microsoft Agent Framework in AI Foundry.

**VS Code workflow:** AI Toolkit → *Create Agent* → Microsoft Agent Framework template.  
Recommended model: `gpt-4o` for all agents.

#### Agent 1 — Extraction Agent
| | |
|---|---|
| **Input** | Raw Cherwell ticket (JSON from Blob) |
| **Prompt** | *"Extract key details from the ticket: description, resolution, work notes, and attachment content."* |
| **Output** | Structured extraction object |

#### Agent 2 — Classification Agent
| | |
|---|---|
| **Input** | Output from Extraction Agent |
| **Prompt** | *"Categorize this issue: Identify Service. Assign Issue Type."* |
| **Output** | `{ "service": "...", "issueType": "..." }` |

#### Agent 3 — Summarization Agent
| | |
|---|---|
| **Input** | Extracted + classified data |
| **Prompt** | *"Summarize the solution. List the steps to resolve this issue."* |
| **Output** | Step-by-step resolution summary |

#### Agent 4 — Validation Agent
| | |
|---|---|
| **Input** | Summary from Summarization Agent |
| **Prompt** | *"Verify the solution: Is it clear, accurate, and free of sensitive information?"* |
| **Output** | Validated article draft or rejection with reason |
| **Key function** | PII scrubbing (names, IPs, credentials) |

#### Agent 5 — Publishing Agent
| | |
|---|---|
| **Input** | Validated article |
| **Prompt** | *"Publish this as a knowledge base article in SharePoint."* |
| **Actions** | Format as Markdown/HTML, add metadata, call SharePoint Graph API |

**Orchestration:** Sequential hand-off — each agent's output is the next agent's input, coordinated by an orchestrator Azure Function or Foundry multi-agent flow.

---

### Phase 4 — SharePoint Knowledge Base Integration

> Publishing Agent writes articles to the SharePoint IT Knowledge Base list.

**SharePoint List: IT Knowledge Base**

| Column | Type |
|--------|------|
| Title | Single line of text |
| Service | Choice |
| IssueType | Choice |
| ResolutionSteps | Multi-line text |
| SourceTicketId | Single line of text |
| AIGenerated | Yes/No |
| PublishedDate | Date |

**Setup steps:**
1. Provision list using PnP.PowerShell or Microsoft Graph API
2. Register an Entra ID app with `Sites.ReadWrite.All` permission
3. Store credentials in Key Vault; Publishing Agent calls `POST /sites/{site-id}/lists/{list-id}/items`

---

### Phase 5 — Azure AI Search Integration

> Semantic retrieval over the knowledge base.

1. Configure AI Search indexer to crawl Blob Storage (`/curated/tickets`)
2. Enable **semantic ranking** and **vector search** on `description` and `resolutionSteps` fields
3. Optionally surface search via a SharePoint web part or a simple REST API

---

### Phase 6 — Monitoring & Observability

> End-to-end visibility with Azure Monitor and Application Insights.

1. Instrument the Azure Function with App Insights SDK (track ingestion counts, failures)
2. Configure AI Foundry to send agent traces to App Insights
3. Set up Log Analytics alerts for:
   - Ingestion failures (Cherwell API errors)
   - Agent validation rejections (PII detected)
   - SharePoint publish failures
4. Use AI Toolkit **Evaluation** to monitor agent quality over time

---

### Phase 7 — Evaluation & Iteration

> Continuously improve agent quality using Foundry's eval loop.

1. Run **batch evaluation** scoring agent outputs against ground-truth resolved tickets
2. Track metrics: accuracy, clarity, PII-free rate
3. Use AI Toolkit **prompt optimization** to refine agent prompts based on eval results
4. Version agent deployments via `agent.yaml`
5. Harvest production traces into evaluation datasets for regression detection

---

## Recommended VS Code Extensions

| Extension | Purpose |
|-----------|---------|
| AI Toolkit (`ms-windows-ai-studio`) | Scaffold, run, and evaluate Foundry agents |
| Azure Functions | Develop & deploy the ingestion function |
| Azure Resources | Browse and manage Azure resources |
| REST Client | Test Cherwell API + Graph API locally |
| Bicep | Author infrastructure as code |

---

## Suggested Build Order

```
Phase 1 (Infra)
  └─ Phase 2 (Ingest Function)
       └─ Phase 3 (Agents — one at a time, validate each before next)
            └─ Phase 4 (SharePoint Publish)
                 └─ Phase 5 (AI Search)
                      └─ Phase 6 (Monitoring)
                           └─ Phase 7 (Eval Loop)
```

> **Tip:** Use a single resolved Cherwell ticket as your end-to-end test case before scaling to batch ingestion.

---

## Repository Structure (Target)

```
/
├── infra/                  # Bicep templates for all Azure resources
├── functions/
│   ├── data_extraction/    # Cherwell API pull
│   └── data_normalization/ # Canonical model transform
├── agents/
│   ├── extraction/         # Agent 1 — agent.yaml + prompt
│   ├── classification/     # Agent 2
│   ├── summarization/      # Agent 3
│   ├── validation/         # Agent 4
│   └── publishing/         # Agent 5
├── sharepoint/
│   └── provision-list.ps1  # PnP PowerShell list provisioning
├── search/
│   └── indexer-config.json # AI Search indexer definition
└── README.md
```

---

## Contributing

This is a hackathon project. Open issues or PRs for any phase. Tag work items with the phase number (e.g., `phase-3-agents`).
