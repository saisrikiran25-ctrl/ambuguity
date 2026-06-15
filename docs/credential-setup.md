# Credential Setup

## Spec Ambiguity Detector — Secure Credential Configuration

This document explains how to add your AI provider credentials to n8n without storing secrets in the workflow file or in this repository.

---

## Important: No Credentials Are Included

The file `workflow/spec-ambiguity-detector.json` contains **no API keys, no bearer tokens, and no secrets** of any kind. All credential references in the workflow JSON are placeholders that point to named credential entries in your n8n instance.

Do not add API keys to the workflow JSON file. Do not commit API keys to source control. Do not share workflow JSON exports that contain credentials.

---

## How n8n Handles Credentials

n8n stores credentials in its own encrypted credential store, separate from workflow definitions. When a workflow node makes an HTTP request, n8n injects the stored credential at runtime — the workflow JSON itself only contains a reference to the credential name, not the credential value.

This means:
- You can safely share or publish the `spec-ambiguity-detector.json` file — it contains no secrets
- Each team member or deployment environment configures their own credentials
- Rotating an API key requires updating the n8n credential store entry, not the workflow

---

## Step 1 — Create the Credential in n8n

1. In n8n, navigate to **Settings** (gear icon in the left sidebar) → **Credentials**
2. Click **+ Add Credential**
3. In the search box, type `HTTP Header Auth` and select it
4. Fill in the following fields:

| Field | Value |
|---|---|
| **Name** | `OpenAI API Key` or `Anthropic API Key` (your choice — this is just the label) |
| **Header Name** | `Authorization` |
| **Header Value** | `Bearer sk-...` (your actual API key, prefixed with `Bearer `) |

5. Click **Save**

For Anthropic, some versions of the API require a different header:

| Field | Value |
|---|---|
| **Header Name** | `x-api-key` |
| **Header Value** | Your Anthropic API key (no `Bearer` prefix) |

If using Anthropic, also set the `anthropic-version` header in the HTTP Request node directly (not in the credential), using the value `2023-06-01` or the current stable version.

---

## Step 2 — Assign the Credential to Workflow Nodes

After creating the credential:

1. Open the `spec-ambiguity-detector` workflow in the n8n editor
2. Click the **AI: Analyze Spec Ambiguity** node
3. In the node panel, find the **Authentication** or **Credential** field
4. Select the credential you created from the dropdown
5. Repeat for the **AI: Portfolio Review** node
6. Save the workflow

Both AI nodes must have the credential assigned before the workflow can make successful AI calls.

---

## Step 3 — Verify the Credential Works

To confirm your credential is configured correctly:

1. In the workflow editor, click **Test Workflow**
2. Send a small test payload through the webhook (use the test URL shown in the Webhook node)
3. Watch the execution flow in the n8n UI
4. If the AI node turns red, click it to see the error message

Common authentication errors:

| Error | Cause | Fix |
|---|---|---|
| `401 Unauthorized` | API key is invalid or the Bearer prefix is missing | Check the Header Value format in your credential |
| `403 Forbidden` | Your API key does not have access to the selected model | Check your API tier or select a model available on your plan |
| `429 Too Many Requests` | Rate limit exceeded | Reduce request frequency or upgrade your API plan |
| Connection refused | Wrong endpoint URL | Verify `_debug_provider_endpoint` in the Provider Config node |

---

## Example Auth Header Formats

**OpenAI**:
```
Header Name:  Authorization
Header Value: Bearer sk-proj-abc123...
```

**Anthropic**:
```
Header Name:  x-api-key
Header Value: sk-ant-api03-abc123...
```

**Azure OpenAI** (if using Azure-hosted models):
```
Header Name:  api-key
Header Value: your-azure-api-key
```
Note: Azure OpenAI also requires a different endpoint format. Consult the Azure OpenAI documentation for the correct base URL.

---

## Rotating or Replacing Credentials

To rotate an API key:

1. Generate a new API key in your provider's dashboard
2. In n8n, navigate to **Settings → Credentials**
3. Find and click your existing credential
4. Update the **Header Value** field with the new key
5. Save

No changes to the workflow JSON are needed. The workflow will automatically use the updated credential on the next execution.

---

## Multi-Environment Setup

If you run multiple n8n environments (e.g., staging and production):

- Create separate API credentials in each environment
- Use separate AI provider keys for staging and production if your provider allows it (helps with usage tracking and rate limit isolation)
- Do not copy credential configurations between environments by exporting and importing workflow JSON — create credentials directly in each environment

---

## What the Workflow JSON Stores

The `spec-ambiguity-detector.json` file stores only credential **references** — the name of the credential to use, not its value. Example of what the credential reference looks like in the raw JSON:

```json
{
  "authentication": "headerAuth",
  "genericAuthType": "httpHeaderAuth",
  "nodeCredentialType": "httpHeaderAuth"
}
```

The actual API key is never written to the JSON. This is n8n's standard credential isolation behavior.

---

## Security Checklist

- [ ] API key stored only in n8n credential store, never in workflow JSON
- [ ] Workflow JSON committed to source control without any secrets
- [ ] Credential name is descriptive and environment-specific
- [ ] API key has been scoped to the minimum required permissions at the provider
- [ ] A plan exists for key rotation if the key is exposed
- [ ] n8n instance itself is behind authentication (do not expose n8n publicly without auth)
