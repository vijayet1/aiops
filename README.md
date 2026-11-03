# AIOps Architecture and Workflows

## Purpose and Scope

This repository contains architecture and workflow documentation for an AIOps platform that integrates large language models (LLMs) into infrastructure investigation. The described architecture separates AI-assisted investigation from state-changing workflows and requires human approval before changes.

This repository documents the platform architecture and workflows; component implementation and deployment instructions are outside its scope.

## Architecture Overview

The architecture is divided into two primary components:

1. **`aiops-dashboard`**: Operations console and API gateway.

2. **`azure-ai-diagnostic-agent`**: Standalone investigation assistant for telemetry and configuration querying.

### Component Overview

```mermaid
flowchart LR
    Operator[Operator]
    Events[Infrastructure and backup failure feed]
    subgraph Dashboard[aiops-dashboard]
        UI[React UI]
        Gateway[API gateway]
        Feedback[Feedback and resolution backend]
        Preview[Browser-memory automation preview]
    end
    Agent[azure-ai-diagnostic-agent]
    ITSM[ServiceNow / ITSM]
    Change[External change agent / services]
    Foundry[Azure AI Foundry]
    Azure[Azure Log Analytics / Microsoft Graph / ARM]

    Operator -->|Use console| UI
    Events -->|Failure events via SSE| UI
    Events -->|Incoming failures for reconciliation| Feedback
    UI -->|Analysis, feedback, and ticket requests| Gateway
    Gateway -->|Record identity-backed confirmations| Feedback
    Gateway <-->|Investigation requests and results| Agent
    Gateway <-->|Retrieve tickets| ITSM
    UI -->|Operator opens with ticket reference| Change
    Gateway -->|Forward operator token for authorization| Change
    UI -->|Simulated steps only| Preview
    Operator <-->|Standalone investigation requests and results| Agent
    Agent <-->|Query generation and correction context| Foundry
    Agent <-->|Read-only requests and evidence via application identity| Azure
```

The component names are architectural identifiers, not directories in this repository. The overview shows logical responsibilities, not a deployment topology. The failure feed's implementation and the external change service are not specified here. ServiceNow supplies ticket data; it is not the destination of the change-agent handoff. The browser-memory preview has no infrastructure execution path.

### Investigation Flow

```mermaid
sequenceDiagram
    actor Operator
    participant Dashboard as Dashboard UI / gateway
    participant Agent as Diagnostic agent backend
    participant Foundry as Azure AI Foundry
    participant Azure as Azure read APIs

    alt Investigation from the console
        Operator->>Dashboard: Request analysis for an event
        Dashboard->>Agent: Forward investigation request
    else Standalone investigation
        Operator->>Agent: Submit operational question
    end
    Agent->>Foundry: Question and Markdown skill schemas
    Foundry-->>Agent: Generated query
    Agent->>Agent: Validate query and execution restrictions
    Agent->>Azure: Execute read-only request using application identity
    Azure-->>Agent: Evidence or query error
    opt Query failure requiring correction
        Note over Agent,Foundry: At most two correction retries
        Agent->>Foundry: Query error and correction context
        Foundry-->>Agent: Corrected query
        Agent->>Agent: Revalidate corrected query
        Agent->>Azure: Retry read-only request
        Azure-->>Agent: Evidence or query error
    end
    opt Cross-source refinement
        Note over Agent,Azure: Up to three additional query steps, subject to the same guardrails
        Agent->>Foundry: Result previews and required dependencies
        Foundry-->>Agent: Follow-up query
        Agent->>Agent: Validate follow-up query
        Agent->>Azure: Execute read-only follow-up
        Azure-->>Agent: Additional evidence
    end
    Agent->>Agent: Correlate evidence and apply required identity filtering
    Note over Agent: Suppress unverified results if required filtering fails
    alt Console request
        Agent-->>Dashboard: Structured results or failure
        Dashboard-->>Operator: Display investigation outcome
    else Standalone request
        Agent-->>Operator: Structured results or failure
    end
```

The correction and refinement blocks summarize bounded behavior; they do not imply unlimited retries. Query generation does not grant the model direct Azure access, and investigation results do not trigger a change.

### Ticket-Based Change Flow

```mermaid
sequenceDiagram
    actor Operator
    participant UI as Dashboard UI
    participant Gateway as Dashboard gateway
    participant ITSM as ServiceNow / ITSM
    participant Change as External change agent / services

    Operator->>UI: Request ITSM tickets
    UI->>Gateway: Retrieve tickets
    Gateway->>ITSM: Request ticket data
    ITSM-->>Gateway: Tickets and requested changes
    Gateway-->>UI: Display ticket details
    Operator->>UI: Open change agent for selected ticket
    UI->>Change: Open with ticket reference prefilled
    Gateway->>Change: Forward operator token for downstream validation
    Change->>Change: Validate token and authorize requested operation
    Note over Operator,Change: Human approval is required before state changes
    Note over Change: External services own mutation and verification
```

The token-forwarding interaction is a logical authorization boundary; the exact launch protocol, approval mechanism, mutation target, and verification API are not specified here. The diagnostic agent is not the change executor.

### Asynchronous Access Inventory Flow

```mermaid
sequenceDiagram
    participant Client as Requesting client
    participant Agent as Diagnostic agent backend
    participant Job as Deterministic Python inventory job
    participant Azure as Azure subscription and identity APIs

    Client->>Agent: Request tenant-wide RBAC inventory
    Agent->>Job: Start background inventory
    Agent-->>Client: Polling ID
    Job->>Azure: Read subscription RBAC and application identities
    Azure-->>Job: Inventory evidence
    Job->>Agent: Make job status and results available
    Client->>Agent: Poll using ID
    Agent-->>Client: Current status or completed inventory
```

This path bypasses the LLM. Job state is process-local as described below; durable recovery and cross-instance polling are not defined.

## Component 1: Operations Console (`aiops-dashboard`)

A central console for monitoring infrastructure events, requesting AI analysis, recording feedback, and handing off approved work to external services.

### Core Workflows

* **Investigate an Operational Event**

  * **Flow:** Infrastructure and backup failures stream to the UI via Server-Sent Events (SSE). The operator requests analysis for a selected event. The API gateway proxies this request to downstream agents and returns the response.

  * **Outcome:** Provides investigation data. Analysis operations do not execute remediation.

* **Record Feedback and Track Resolution**

  * **Flow:** Operator confirms an AI recommendation. The gateway stamps the request with validated Entra ID token claims. The backend reconciles shared confirmations with incoming failures to triage recurring issues.

  * **Outcome:** Identity-backed tracking of operator actions. Distinguishes resolved incidents from active failures.

* **Hand Off a Ticket-Based Change**

  * **Flow:** The console retrieves ITSM tickets (e.g., ServiceNow) and displays requested network rule changes. The operator opens the change agent with the ticket reference prefilled. The gateway forwards the operator's token for downstream RBAC validation.

  * **Outcome:** Connects a ticket to a change workflow. External services handle actual state mutation and verification.

* **Preview an Automation Run (simulated)**

  * **Flow:** The UI launches a simulated automation candidate. The browser-memory run displays preflight, approval, execution, and verification steps.

  * **Outcome:** Validates user interface workflow. Displayed checks do not execute Azure infrastructure actions.

## Component 2: Diagnostic Agent (`azure-ai-diagnostic-agent`)

An investigation assistant that translates natural-language queries into executable syntax, retrieves operational data, and returns structured results.

### Core Workflows

* **Retrieve Evidence via Query Generation**

  * **Flow:** User submits an operational question. Markdown skills supply schemas to the LLM (Azure AI Foundry). The LLM generates a query. The backend executes it using the application's managed identity.

  * **Required guardrails:** Azure Resource Manager (ARM) and Microsoft Graph executors must enforce **GET-only** requests and allowed hosts. Read-only access must be enforced by executor validation and the application's granted permissions, not by model instructions alone.

* **Self-Correct Failed Queries**

  * **Flow:** If a generated KQL or Graph query fails execution, the backend feeds the error back to the model as correction context.

  * **Described limit:** Correction routines allow at most **two retries**. This limit must be verified against the component implementation.

* **Correlate Sources and Refine Results**

  * **Flow:** The agent executes an initial query and feeds result previews back to the model. Cross-source dependencies can trigger up to **three additional query steps** in the described workflow. Follow-up queries remain subject to the same execution guardrails.

  * **Outcome:** Merges evidence across multiple infrastructure domains. The backend suppresses unverified results if required identity filtering fails.

* **Long-Running Access Inventory**

  * **Flow:** A request for a tenant-wide RBAC inventory bypasses the LLM. The backend triggers a deterministic Python job that scans subscriptions and enriches application identities, returning a polling ID to the client.

  * **Outcome:** Asynchronous reporting workflow decoupled from LLM generation.

## System Boundaries & Security

The following are security and operational requirements for the described architecture; this repository does not contain code that enforces them:

* **Identity & Authentication:** Use MSAL/Entra ID for authentication. Downstream APIs must validate bearer-token signature, issuer, audience, and expiry and enforce authorization. Token forwarding alone does not establish RBAC. Any local-development authentication bypass must be explicitly isolated from deployed environments; missing tenant/app configuration must fail closed in deployed environments.

* **User Context vs. Execution Identity:** The gateway's forwarded user token and the diagnostic agent's application managed identity are distinct authorization boundaries. User authentication does not by itself constrain the managed identity's access. Grant the application only the read permissions needed for supported investigations and authorize access to the returned evidence.

* **Process-Local State:** Conversation history and background jobs are described as process-local and lost on restart. Treat polling IDs as instance-local; restart recovery and cross-instance job lookup are not defined here. Process-local state is not a guarantee that logs, external services, or query results leave no persistent data.

* **Read-Only AI Authorization:** Query results are investigation evidence, not triggers for automated remediation. Enforce read-only execution outside the LLM. Human-approved changes belong to external change workflows, whose mutation permissions and verification controls must be assessed separately.

## Described Technology Stack

* **Frontend:** React, TypeScript, Server-Sent Events (SSE)

* **Backend:** Python, FastAPI, Pydantic

* **AI/ML:** Azure AI Foundry, LLM Orchestration, Prompt Engineering (Markdown Skills)

* **Cloud Infrastructure:** Microsoft Entra ID (MSAL), Azure Resource Manager (ARM), Microsoft Graph API, Azure Log Analytics (KQL)
