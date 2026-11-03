# AIOps Architecture and Workflows

## Purpose and Scope

This repository documents the architecture and workflows of an AIOps platform. The system integrates Large Language Models (LLMs) into infrastructure management while enforcing strict human-in-the-loop boundaries for all execution paths.

The architecture is divided into two primary components:

1. **`aiops-dashboard`**: Operations console and API gateway.

2. **`azure-ai-diagnostic-agent`**: Standalone investigation assistant for telemetry and configuration querying.

```
graph LR
    Operator -->|React UI| Dashboard[aiops-dashboard]
    Dashboard -->|API Gateway| Agent[azure-ai-diagnostic-agent]
    Dashboard -->|Ticket Handoff| ITSM[ServiceNow]
    Agent -->|Generated Queries| Azure[Azure Log Analytics / Graph / ARM]

```

## Component 1: Operations Console (`aiops-dashboard`)

A central console for monitoring infrastructure events, requesting AI analysis, recording feedback, and handing off approved work to external services.

### Core Workflows

* **Investigate an Operational Event (ACTIVE)**

  * **Flow:** Infrastructure and backup failures stream to the UI via Server-Sent Events (SSE). The operator requests analysis for a selected event. The API gateway proxies this request to downstream agents and returns the response.

  * **Outcome:** Provides investigation data. Analysis operations do not execute remediation.

* **Record Feedback and Track Resolution (ACTIVE)**

  * **Flow:** Operator confirms an AI recommendation. The gateway stamps the request with validated Entra ID token claims. The backend reconciles shared confirmations with incoming failures to triage recurring issues.

  * **Outcome:** Identity-backed tracking of operator actions. Distinguishes resolved incidents from active failures.

* **Hand Off a Ticket-Based Change (ACTIVE)**

  * **Flow:** The console retrieves ITSM tickets (e.g., ServiceNow) and displays requested network rule changes. The operator opens the change agent with the ticket reference prefilled. The gateway forwards the operator's token for downstream RBAC validation.

  * **Outcome:** Connects a ticket to a change workflow. External services handle actual state mutation and verification.

* **Preview an Automation Run (SIMULATED)**

  * **Flow:** The UI launches a simulated automation candidate. The browser-memory run displays preflight, approval, execution, and verification steps.

  * **Outcome:** Validates user interface workflow. Displayed checks do not execute Azure infrastructure actions.

## Component 2: Diagnostic Agent (`azure-ai-diagnostic-agent`)

An investigation assistant that translates natural-language queries into executable syntax, retrieves operational data, and returns structured results.

### Core Workflows

* **Retrieve Evidence via Query Generation (ACTIVE)**

  * **Flow:** User submits an operational question. Markdown skills supply schemas to the LLM (Azure AI Foundry). The LLM generates a query. The backend executes it using the application's managed identity.

  * **Guardrails:** Azure Resource Manager (ARM) and Microsoft Graph executors strictly enforce **GET-only** requests and allowed hosts. The LLM does not possess write access.

* **Self-Correct Failed Queries (ACTIVE)**

  * **Flow:** If a generated KQL or Graph query fails execution, the backend feeds the error back to the model as correction context.

  * **Guardrails:** Correction routines are hard-capped at **two retries** to prevent infinite loops.

* **Correlate Sources and Refine Results (ACTIVE)**

  * **Flow:** The agent executes an initial query and feeds result previews back to the model. Cross-source dependencies trigger up to three additional query steps.

  * **Outcome:** Merges evidence across multiple infrastructure domains. The backend suppresses unverified results if required identity filtering fails.

* **Long-Running Access Inventory (ACTIVE)**

  * **Flow:** A request for a tenant-wide RBAC inventory bypasses the LLM. The backend triggers a deterministic Python job that scans subscriptions and enriches application identities, returning a polling ID to the client.

  * **Outcome:** Asynchronous reporting workflow decoupled from LLM generation.

## System Boundaries & Security

The platform enforces the following operational constraints:

* **Identity & Authentication:** Authentication utilizes MSAL/Entra ID. The gateway forwards bearer tokens to downstream services to preserve user context for RBAC. Missing tenant/app configuration bypasses token validation strictly for local development.

* **Stateless Operations:** Conversation history and background jobs are process-local and lost on restart. Ephemeral execution prevents state persistence across sessions.

* **Read-Only AI Authorization:** Data access is mediated by the application's identity. LLM operations are restricted to read execution. Query results function as investigation evidence, not as triggers for automated remediation.

## Technology Stack

* **Frontend:** React, TypeScript, Server-Sent Events (SSE)

* **Backend:** Python, FastAPI, Pydantic

* **AI/ML:** Azure AI Foundry, LLM Orchestration, Prompt Engineering (Markdown Skills)

* **Cloud Infrastructure:** Microsoft Entra ID (MSAL), Azure Resource Manager (ARM), Microsoft Graph API, Azure Log Analytics (KQL)