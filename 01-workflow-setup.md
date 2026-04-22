# 01 — Workflow Setup

## Overview
This workflow extends the `wazuh-n8n-soar` pipeline by adding a Claude AI
analysis node between the threat intelligence enrichment and the Discord
notification. Claude acts as a first-line SOC analyst, triaging each alert
and generating a human-readable severity assessment and recommended action.

## Workflow Architecture

```
Webhook → AbuseIPDB Enrichment → VirusTotal Enrichment → Claude AI Analysis → Threat Filter → Discord Notification
                                                                                     ↓ false
                                                                              Log Ignored Alert
```

## Prerequisites

- n8n running at `https://172.20.70.194`
- Anthropic API key from console.anthropic.com
- AbuseIPDB API key from abuseipdb.com
- VirusTotal API key from virustotal.com
- Discord webhook URL

## Step 1 — Create a new workflow in n8n

1. Open n8n at `https://172.20.70.194`
2. Click **"New workflow"**
3. Rename it to: `Wazuh Claude SOC Analyst`

## Step 2 — Add Webhook trigger node

- **Type:** Webhook
- **HTTP Method:** POST
- **Path:** `wazuh-claude-alert`
- **Authentication:** None
- **Respond:** Immediately

Production URL:
```
https://YOUR_N8N_IP/webhook/wazuh-claude-alert
```

> Note: Using a different path from the previous workflow (`wazuh-alert`)
> to keep both workflows active simultaneously.

## Step 3 — Add AbuseIPDB Enrichment node

- **Type:** HTTP Request
- **Method:** GET
- **URL:** `https://api.abuseipdb.com/api/v2/check`
- **Settings:** Enable "Continue on Fail"

Query Parameters:

| Name | Value |
|------|-------|
| `ipAddress` | `{{ $json.body.src_ip }}` |
| `maxAgeInDays` | `90` |

Headers:

| Name | Value |
|------|-------|
| `Accept` | `application/json` |
| `Key` | `YOUR_ABUSEIPDB_API_KEY` |

## Step 4 — Add VirusTotal Enrichment node

- **Type:** HTTP Request
- **Method:** GET
- **URL:** `https://www.virustotal.com/api/v3/ip_addresses/{{ $('Webhook').item.json.body.src_ip }}`
- **Settings:** Enable "Continue on Fail"

Headers:

| Name | Value |
|------|-------|
| `x-apikey` | `YOUR_VIRUSTOTAL_API_KEY` |

## Step 5 — Add Claude AI Analysis node

See [02 — Claude AI Node](02-claude-ai-node.md) for full configuration.

## Step 6 — Add Threat Filter node

- **Type:** IF
- **Condition:** `{{ $('Claude AI Analysis').item.json.severity }}` not equal to `LOW`

> Filters out low severity alerts — only MEDIUM, HIGH and CRITICAL
> alerts reach the Discord notification.

## Step 7 — Add Discord Notification node

- **Type:** HTTP Request
- **Method:** POST
- **URL:** `YOUR_DISCORD_WEBHOOK_URL`
- **Body Content Type:** JSON

```json
{
  "embeds": [{
    "title": "🚨 Wazuh Security Alert",
    "color": 15158332,
    "fields": [
      {
        "name": "📋 Rule",
        "value": "{{ $('Webhook').item.json.body.rule_desc }}",
        "inline": false
      },
      {
        "name": "🌐 Source IP",
        "value": "{{ $('Webhook').item.json.body.src_ip }}",
        "inline": true
      },
      {
        "name": "⚠️ Abuse Score",
        "value": "{{ $('AbuseIPDB Enrichment').item.json.data.abuseConfidenceScore }}%",
        "inline": true
      },
      {
        "name": "🦠 VT Malicious Votes",
        "value": "{{ $('VirusTotal Enrichment').item.json.data.attributes.total_votes.malicious }}",
        "inline": true
      },
      {
        "name": "🤖 AI Severity",
        "value": "{{ $('Claude AI Analysis').item.json.severity }}",
        "inline": true
      },
      {
        "name": "🎯 Recommendation",
        "value": "{{ $('Claude AI Analysis').item.json.recommendation }}",
        "inline": false
      },
      {
        "name": "🧠 AI Analysis",
        "value": "{{ $('Claude AI Analysis').item.json.analysis }}",
        "inline": false
      },
      {
        "name": "🖥️ Agent",
        "value": "{{ $('Webhook').item.json.body.agent_name }}",
        "inline": true
      },
      {
        "name": "🕐 Timestamp",
        "value": "{{ $('Webhook').item.json.body.timestamp }}",
        "inline": true
      }
    ],
    "footer": {
      "text": "Wazuh + Claude AI SOC Analyst | n8n Automation"
    }
  }]
}
```

## Step 8 — Add Log Ignored Alert node (false branch)

- **Type:** Code (JavaScript)
- **Branch:** Threat Filter → false

```javascript
const alert = $('Webhook').item.json.body;
const severity = $('Claude AI Analysis').item.json.severity;

console.log(`[IGNORED] ${alert.timestamp} | Severity: ${severity} | Rule: ${alert.rule_id} | IP: ${alert.src_ip} | Agent: ${alert.agent_name}`);

return {
  status: "ignored",
  severity: severity,
  reason: "below_severity_threshold",
  rule_id: alert.rule_id,
  src_ip: alert.src_ip,
  agent_name: alert.agent_name,
  timestamp: alert.timestamp
};
```

## Step 9 — Publish workflow

Click **"Publish"** at the top right to activate the workflow in production mode.

## Screenshots
![Workflow canvas](../screenshots/02-n8n-workflow/workflow-canvas.png)
![Webhook node](../screenshots/02-n8n-workflow/webhook-node.png)
