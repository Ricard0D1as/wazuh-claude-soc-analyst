# 03 — Discord Output & Testing

## Overview
The Discord notification now includes Claude's AI analysis alongside
the raw threat intelligence data — transforming a technical alert into
an actionable, human-readable security report.

## Discord Alert Format

```
🚨 Wazuh Security Alert

📋 Rule
File added to /root directory

🌐 Source IP          ⚠️ Abuse Score     🦠 VT Malicious
185.220.101.45        100%               11

🤖 AI Severity
CRITICAL

🎯 Recommendation
Block this IP immediately at the firewall level. Investigate all
recent connections from this IP and check for successful auth attempts.

🧠 AI Analysis
This IP is a confirmed Tor exit node with a 100% AbuseIPDB confidence
score and 11 malicious VirusTotal votes. It has been actively used for
SSH brute-force attacks across multiple targets as recently as April 2026.

🖥️ Agent             🕐 Timestamp
Debian13GreenDPR     2026-04-15T10:00:00

Wazuh + Claude AI SOC Analyst | n8n Automation
```

## Severity Color Coding

The Discord embed color changes based on AI severity:

| Severity | Color | Discord Color Code |
|----------|-------|--------------------|
| LOW | Green | 3066993 |
| MEDIUM | Yellow | 16776960 |
| HIGH | Orange | 16744272 |
| CRITICAL | Red | 15158332 |

To implement dynamic colors, update the Discord node body:

```json
{
  "embeds": [{
    "color": "{{ $('Claude AI Analysis').item.json.severity === 'CRITICAL' ? 15158332 : $('Claude AI Analysis').item.json.severity === 'HIGH' ? 16744272 : $('Claude AI Analysis').item.json.severity === 'MEDIUM' ? 16776960 : 3066993 }}"
  }]
}
```

## Testing

### Test 1 — Known malicious IP (should reach Discord)

```bash
curl -k -X POST https://YOUR_N8N_IP/webhook-test/wazuh-claude-alert \
  -H "Content-Type: application/json" \
  -d '{
    "rule_id":"100201",
    "rule_desc":"File added to /root directory",
    "level":7,
    "src_ip":"185.220.101.45",
    "agent_name":"Debian13GreenDPR",
    "timestamp":"2026-04-15T10:00:00"
  }'
```

Expected result:
- AbuseIPDB Score: 100%
- VT Malicious: 11
- AI Severity: CRITICAL
- Discord notification: ✅

### Test 2 — Clean private IP (should be logged and ignored)

```bash
curl -k -X POST https://YOUR_N8N_IP/webhook-test/wazuh-claude-alert \
  -H "Content-Type: application/json" \
  -d '{
    "rule_id":"100200",
    "rule_desc":"File modified in /root directory",
    "level":7,
    "src_ip":"10.0.0.1",
    "agent_name":"Debian13GreenDPR",
    "timestamp":"2026-04-15T10:00:00"
  }'
```

Expected result:
- AbuseIPDB Score: 0%
- AI Severity: LOW
- Discord notification: ❌ (filtered)
- Log Ignored Alert: ✅

### Test 3 — Real Wazuh alert (production test)

On the Wazuh Manager VM:

```bash
sudo touch /root/suspicious_file.exe
```

Expected result:
- Wazuh detects file addition in /root (rule 100201)
- n8n receives alert via webhook
- AbuseIPDB + VirusTotal enrich the alert
- Claude AI generates triage analysis
- If severity > LOW: Discord notification sent

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Claude returns non-JSON | Model generated markdown | Check prompt — ensure "no markdown, no backticks" instruction is present |
| Parse AI Response fails | Claude API error | Enable "Continue on Fail" on Claude node |
| Discord shows "undefined" | Wrong node reference | Check exact node names are case-sensitive |
| AbuseIPDB rate limit | Too many requests | Enable "Continue on Fail", restrict Wazuh integration to specific rule_ids |

## Screenshots
![Discord alert CRITICAL](../screenshots/04-final-result/discord-critical.png)
![Discord alert LOW ignored](../screenshots/04-final-result/discord-ignored.png)
![Full workflow execution](../screenshots/04-final-result/full-execution.png)
