# 02 — Claude AI Analysis Node

## Overview
The Claude AI Analysis node is the core of this project. It receives
enriched alert data from AbuseIPDB and VirusTotal, sends a structured
prompt to Claude Haiku 4.5, and returns a JSON object containing:
- A natural language threat analysis
- A severity classification (LOW/MEDIUM/HIGH/CRITICAL)
- A recommended response action

## Node Configuration

- **Type:** HTTP Request
- **Method:** POST
- **URL:** `https://api.anthropic.com/v1/messages`
- **Settings:** Enable "Continue on Fail"

## Headers

| Name | Value |
|------|-------|
| `x-api-key` | `YOUR_ANTHROPIC_API_KEY` |
| `anthropic-version` | `2023-06-01` |
| `content-type` | `application/json` |

## Body Content Type
`JSON`

## Body

```json
{
  "model": "claude-haiku-4-5",
  "max_tokens": 1024,
  "messages": [
    {
      "role": "user",
      "content": "You are a SOC analyst. Analyze this security alert and respond ONLY with a valid JSON object (no markdown, no backticks) with exactly these fields: analysis (string), severity (one of: LOW, MEDIUM, HIGH, CRITICAL), recommendation (string).\n\nAlert data:\n- Rule: {{ $('Webhook').item.json.body.rule_desc }}\n- Source IP: {{ $('Webhook').item.json.body.src_ip }}\n- Agent: {{ $('Webhook').item.json.body.agent_name }}\n- Timestamp: {{ $('Webhook').item.json.body.timestamp }}\n- AbuseIPDB Score: {{ $('AbuseIPDB Enrichment').item.json.data.abuseConfidenceScore }}%\n- AbuseIPDB Total Reports: {{ $('AbuseIPDB Enrichment').item.json.data.totalReports }}\n- AbuseIPDB Country: {{ $('AbuseIPDB Enrichment').item.json.data.countryCode }}\n- AbuseIPDB ISP: {{ $('AbuseIPDB Enrichment').item.json.data.isp }}\n- VirusTotal Malicious Votes: {{ $('VirusTotal Enrichment').item.json.data.attributes.total_votes.malicious }}\n- VirusTotal Harmless Votes: {{ $('VirusTotal Enrichment').item.json.data.attributes.total_votes.harmless }}\n- VirusTotal Tags: {{ $('VirusTotal Enrichment').item.json.data.attributes.tags }}"
    }
  ]
}
```

## Post-processing — Extract AI Response

After the HTTP Request node, add a **Code node** to parse the Claude response:

- **Type:** Code (JavaScript)
- **Name:** `Parse AI Response`

```javascript
// Get the raw response from Claude API
const response = $input.item.json;

// Extract the text content from Claude's response
const rawText = response.content[0].text;

// Parse the JSON response from Claude
try {
  const parsed = JSON.parse(rawText);
  return {
    analysis: parsed.analysis || "Analysis unavailable",
    severity: parsed.severity || "UNKNOWN",
    recommendation: parsed.recommendation || "No recommendation available",
    raw_response: rawText
  };
} catch (e) {
  // If parsing fails, return raw text with default severity
  return {
    analysis: rawText,
    severity: "UNKNOWN",
    recommendation: "Manual review required",
    raw_response: rawText
  };
}
```

## How the Prompt Works

The prompt instructs Claude to:

1. **Act as a SOC analyst** — sets the professional context
2. **Respond only in JSON** — ensures parseable structured output
3. **Use exactly three fields** — analysis, severity, recommendation
4. **Classify severity** using a fixed scale — LOW/MEDIUM/HIGH/CRITICAL

This structured approach ensures the output is always parseable by
the downstream nodes regardless of the alert content.

## Example Claude Response

Input alert (185.220.101.45 — Tor exit node):

```json
{
  "analysis": "This IP address is a confirmed Tor exit node with a 100% AbuseIPDB confidence score and 11 malicious VirusTotal votes. It has been actively used for SSH brute-force attacks across multiple targets as recently as April 2026. The combination of Tor anonymization and active malicious activity makes this a high-confidence threat.",
  "severity": "CRITICAL",
  "recommendation": "Block this IP immediately at the firewall level. Investigate all recent connections from this IP in your environment and check for any successful authentication attempts. Consider implementing Tor exit node blocklists."
}
```

Input alert (10.0.0.1 — internal private IP):

```json
{
  "analysis": "This is a private RFC1918 IP address with no public abuse history. The alert was triggered by a FIM rule monitoring the /root directory. This may be a legitimate administrative action or an internal threat.",
  "severity": "LOW",
  "recommendation": "Verify with the system administrator whether this file operation was authorized. No external blocking action required."
}
```

## Cost Estimation

| Model | Input tokens | Output tokens | Cost per alert |
|-------|-------------|---------------|---------------|
| Claude Haiku 4.5 | ~500 | ~300 | ~$0.0004 |

With $5 in free Anthropic credits: ~12,000 alert analyses.

## Screenshots
![Claude AI node configuration](../screenshots/02-n8n-workflow/claude-node-config.png)
![Claude AI node output](../screenshots/03-ai-analysis/claude-output.png)
![Parse AI Response node](../screenshots/03-ai-analysis/parse-response.png)
