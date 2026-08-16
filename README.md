# 🛡️ AI-Powered Cyber Security Incident Response Platform

An n8n-based automation platform that collects security alerts from multiple tools, classifies and prioritizes them using AI (Groq/Llama 3.3), routes them to the right team with an optional human-approval gate for critical incidents, logs everything to a centralized audit trail, and generates daily analytics reports.


---

## 🎥 Demo

https://drive.google.com/file/d/1SLuIlq_ArzZGf8gFmqB_SCamzRmky0Of/view?usp=drive_link

---

## 📐 Architecture

See [`ARCHITECTURE.md`](./ARCHITECTURE.md) for full diagrams. High-level flow:

```
Security Tool → WF1 (Collect) → WF2 (AI Classify) → WF3 (Assign/Escalate + Human Approval)
             → WF4 (Log + Notify) → Google Sheets + Slack/Gmail

WF5 (Daily Cron) → reads Sheets → emails/Slack analytics report
WF6 (Error Handler) → catches failures from WF1-WF4 → logs + alerts DevOps
```

## 📦 What's in this repo

```
├── workflows/
│   ├── WF1_Security_Alert_Collection.json
│   ├── WF2_AI_Threat_Classification.json
│   ├── WF3_Incident_Assignment_Escalation.json
│   ├── WF4_Resolution_Tracking_Notifications.json
│   ├── WF5_Security_Dashboard_Reports.json
│   └── WF6_Error_Handling_Audit_Trail.json
├── PROBLEM_ANALYSIS.md
├── ARCHITECTURE.md
├── Presentation.pptx
└── README.md   (this file)
```

## ✨ Features

| Category | Implementation |
|---|---|
| **AI-powered decision making** | Groq `llama-3.3-70b-versatile` classifies severity, category, team, confidence, and recommended action (WF2) |
| **Human approval step** | Critical incidents require SOC-lead Slack approval (Approve/Downgrade buttons) before full escalation, with a 30-min auto-resume timeout (WF3) |
| **Error handling & retry** | HTTP call to Groq retries 3x automatically; rule-based fallback classification if AI is unreachable (WF2) |
| **Logging & audit trail** | Every incident (WF4) and every pipeline failure (WF6) is appended to Google Sheets with full timestamps |
| **Scheduled workflows (Cron)** | Daily 8 AM report workflow (WF5) reads the log and emails/Slacks a severity/category/team breakdown |
| **Webhook-triggered workflows** | WF1 is the single external entry point for all security tools |
| **Conditional branching** | Severity-based Switch routing in WF3/WF4; validation IF-gate in WF1 |

## 🚀 Setup

### Prerequisites
- An n8n instance (n8n.cloud or self-hosted)
- A free [Groq API key](https://console.groq.com)
- A Google account (for Sheets + Gmail)
- A Slack workspace with a bot token

### 1. Create the Google Sheet
Create one spreadsheet with two tabs:
- `Security_Incident_Log` — headers: `incidentId, receivedAt, source, alertType, severity, category, assignedTeam, confidence, status, sourceIP, targetIP, aiSummary, recommendedAction, slaMinutes, classificationSource, approvalStatus, resolutionStatus, notifiedAt`
- `Audit_Error_Log` — headers: `failedWorkflow, failedNode, errorMessage, occurredAt`

### 2. Import all 6 workflows
In n8n: **Workflows → Import from File** for each `.json` in `/workflows`. Import all six before wiring connections (step 3 needs every workflow to already exist).

### 3. Reconnect the Execute Workflow links
n8n regenerates workflow IDs on import, so each `Execute Workflow` node needs to be re-pointed:
- WF1 → `Execute Workflow - WF2 Classification` → reselect **WF2**
- WF2 → `Execute Workflow - WF3 Assignment` → reselect **WF3**
- WF3 → `Execute Workflow - WF4 Notifications` → reselect **WF4**

Also set **WF6** as the `Error Workflow` in the Settings tab of WF1, WF2, WF3, and WF4.

### 4. Add credentials
- **Groq**: paste your API key into the Authorization header in WF2's HTTP Request node (or convert to an n8n Header Auth credential)
- **Google Sheets**: connect via OAuth on the Sheets nodes in WF4, WF5, WF6; set the Document ID
- **Slack**: connect a bot with `chat:write` scope; invite it to `#soc-critical-alerts`, `#soc-high-alerts`, `#soc-general`, `#soc-devops-alerts`
- **Gmail**: connect via OAuth on the Gmail nodes in WF4 and WF5; set real recipient addresses

### 5. Activate
Toggle each workflow to **Active**. WF1's webhook URL becomes the production endpoint — point your SIEM/firewall/IDS/AV tools at it.

### 6. Test
```powershell
$body = @{
    source = "Firewall"
    alertType = "Repeated failed SSH login"
    description = "50 failed SSH attempts from external IP in 2 minutes"
    sourceIP = "203.0.113.45"
    targetIP = "10.0.0.12"
} | ConvertTo-Json

Invoke-RestMethod -Uri "https://YOUR-N8N-URL/webhook/security-alert" -Method Post -Body $body -ContentType "application/json"
```

## 🧩 Tech Stack
- **Orchestration**: n8n
- **AI**: Groq (Llama 3.3 70B) — free tier
- **Storage/Audit**: Google Sheets
- **Notifications**: Slack, Gmail

## 📄 License
Built for educational/hackathon purposes.
