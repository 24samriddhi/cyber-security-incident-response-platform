# Workflow Architecture
## AI-Powered Cyber Security Incident Response Platform

### 1. Overall System Architecture

The platform is built as **6 independent, modular n8n workflows** that chain together via `Execute Workflow` calls, plus one cron-scheduled reporting workflow and one always-on error handler. This mirrors microservice design: each workflow has one responsibility, can be tested/deployed independently, and can be swapped out (e.g., replace Groq with another LLM) without touching the rest of the pipeline.

```mermaid
flowchart TB
    subgraph External["External Alert Sources"]
        SIEM[SIEM Tools]
        FW[Firewall]
        IDS[IDS/IPS]
        AV[Antivirus/EDR]
        CLOUD[Cloud Monitoring]
    end

    subgraph Platform["n8n Automation Platform"]
        WF1["WF1: Security Alert Collection\n(Webhook Trigger)"]
        WF2["WF2: AI Threat Classification\n& Prioritization\n(Groq LLM)"]
        WF3["WF3: Incident Assignment\n& Escalation\n(Human Approval Gate)"]
        WF4["WF4: Resolution Tracking\n& Notifications"]
        WF5["WF5: Security Dashboard\n& Incident Reports\n(Cron, Daily)"]
        WF6["WF6: Error Handling\n& Audit Trail\n(Global Error Catcher)"]
    end

    subgraph Storage["Centralized Data Store"]
        SHEET[(Google Sheets\nSecurity_Incident_Log)]
        ERRLOG[(Google Sheets\nAudit_Error_Log)]
    end

    subgraph Notify["Stakeholder Channels"]
        SLACK[Slack Channels]
        GMAIL[Gmail]
    end

    SIEM --> WF1
    FW --> WF1
    IDS --> WF1
    AV --> WF1
    CLOUD --> WF1

    WF1 -->|Execute Workflow| WF2
    WF2 -->|Execute Workflow| WF3
    WF3 -->|Execute Workflow| WF4
    WF4 --> SHEET
    WF4 --> SLACK
    WF4 --> GMAIL

    WF5 --> SHEET
    WF5 --> GMAIL
    WF5 --> SLACK

    WF1 -.on error.-> WF6
    WF2 -.on error.-> WF6
    WF3 -.on error.-> WF6
    WF4 -.on error.-> WF6
    WF6 --> ERRLOG
    WF6 --> SLACK
```

### 2. Workflow Interaction Diagram

Each workflow is triggered either externally (webhook, cron) or internally (Execute Workflow call carrying the incident JSON forward). Data accumulates as it passes through the chain — each workflow adds fields to the same JSON object.

```mermaid
sequenceDiagram
    participant Src as Security Tool
    participant WF1 as WF1 Alert Collection
    participant WF2 as WF2 AI Classification
    participant Groq as Groq LLM API
    participant WF3 as WF3 Assignment/Escalation
    participant Human as SOC Lead (Slack)
    participant WF4 as WF4 Notifications
    participant Sheet as Google Sheets
    participant Chan as Slack/Gmail

    Src->>WF1: POST /security-alert (JSON)
    WF1->>WF1: Validate + Normalize
    WF1->>WF2: Execute Workflow (alert data)
    WF2->>Groq: Classify severity/category/team
    Groq-->>WF2: Structured JSON classification
    WF2->>WF2: Parse (fallback rules if AI fails)
    WF2->>WF3: Execute Workflow (classified incident)
    WF3->>WF3: Switch on severity

    alt Severity = Critical
        WF3->>Human: Send & Wait (Approve/Downgrade)
        Human-->>WF3: Response (or 30-min timeout)
    else High / Medium / Low
        WF3->>WF3: Auto-assign, no approval needed
    end

    WF3->>WF4: Execute Workflow (final incident)
    WF4->>Sheet: Append row (audit trail)
    WF4->>Chan: Notify (Slack/Gmail by severity)
    WF4-->>WF3: return
    WF3-->>WF2: return
    WF2-->>WF1: return
    WF1-->>Src: 200 OK {incidentId, severity, team}
```

### 3. Event Flow Between Workflows

| From | To | Trigger Mechanism | Data Passed |
|---|---|---|---|
| External tool | WF1 | HTTP POST webhook | Raw alert JSON |
| WF1 | WF2 | Execute Workflow (sync, waits for result) | Normalized alert + incidentId |
| WF2 | Groq API | HTTP Request (with retry x3) | Alert description/context |
| WF2 | WF3 | Execute Workflow (sync) | Alert + severity/category/team/SLA |
| WF3 | Slack (human) | Send & Wait (only if Critical) | Approval prompt, 30 min timeout |
| WF3 | WF4 | Execute Workflow (sync) | Final incident record + approval status |
| WF4 | Google Sheets | Append row | Full incident record (audit trail) |
| WF4 | Slack/Gmail | Notification send | Formatted incident summary |
| WF1-WF4 | WF6 | n8n's built-in `errorWorkflow` setting | Failed execution metadata |
| Cron (8 AM daily) | WF5 | Schedule Trigger | — |
| WF5 | Google Sheets | Read all rows | Last 24h of incidents |
| WF5 | Gmail/Slack | Analytics report send | Aggregated counts, top vulnerability |

### 4. Why This Modular Split (Design Rationale)

- **WF1 (Collection)** is the only webhook-facing workflow — a single, controlled entry point simplifies security (one endpoint to authenticate/rate-limit) and input validation.
- **WF2 (Classification)** isolates the AI dependency. If Groq is swapped for another provider, only this workflow changes. It also owns retry logic and a rule-based fallback so the pipeline never hard-fails just because the LLM is down.
- **WF3 (Assignment/Escalation)** owns the human-in-the-loop gate. Keeping approval logic separate from notification logic means the approval UX can change without touching how Slack/Gmail messages are formatted.
- **WF4 (Resolution/Notifications)** is the only workflow that writes to the audit log and sends stakeholder-facing messages — a single place to change notification formatting or add new channels (e.g., MS Teams) later.
- **WF5 (Dashboard/Reports)** is fully decoupled and cron-driven — it can be changed or re-scheduled without any risk to the real-time alert pipeline.
- **WF6 (Error Handling)** is wired in as n8n's global `errorWorkflow` for WF1-WF4, so any node failure anywhere is caught centrally instead of needing duplicate error-handling logic in every workflow.
