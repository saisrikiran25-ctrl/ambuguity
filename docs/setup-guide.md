# Setup Guide

## Spec Ambiguity Detector — Import, Configure, and First Run

This guide walks through getting the workflow running in your n8n instance from scratch. It assumes you have an active n8n instance (self-hosted or n8n Cloud) and an AI provider account (OpenAI or Anthropic).

---

## Prerequisites

| Requirement | Notes |
|---|---|
| n8n instance | Version 1.30 or later recommended. Cloud or self-hosted both supported. |
| AI provider account | OpenAI or Anthropic. Other providers with compatible API shapes will also work. |
| API key | From your AI provider. This must be stored in n8n credentials — not in the workflow file. |
| HTTP client | cURL, Postman, Insomnia, or any tool capable of sending POST requests with JSON bodies. |

---

## Step 1 — Import the Workflow

1. Open your n8n instance in a browser.
2. Navigate to **Workflows** in the left sidebar.
3. Click **+ New Workflow**, then select **Import from File** (or use the three-dot menu on the Workflows page and select **Import**).
4. Select the file `workflow/spec-ambiguity-detector.json` from this repository.
5. n8n will load the workflow. You should see the full node graph rendered.
6. Click **Save** to persist the imported workflow.

**If the import fails**: Confirm you are using the complete `spec-ambiguity-detector.json` file and that your n8n version supports all node types used. The workflow uses only standard n8n node types: Webhook, Set, Code, Switch, HTTP Request, and Respond to Webhook.

---

## Step 2 — Configure Your AI Provider Credential

Before activating the workflow, you must add your AI provider credential to n8n.

1. In n8n, navigate to **Settings → Credentials → + Add Credential**.
2. Select **HTTP Header Auth** as the credential type.
3. Set the **Name** field to something like `OpenAI API Key` or `Anthropic API Key`.
4. Set the **Header Name** field to `Authorization`.
5. Set the **Header Value** field to `Bearer YOUR_API_KEY_HERE` (replace with your actual key).
6. Save the credential.

See [`docs/credential-setup.md`](credential-setup.md) for full details on credential handling.

---

## Step 3 — Configure the Provider Config Node

After importing the workflow:

1. Open the workflow in the n8n editor.
2. Click the **Provider Config** node.
3. Update the following fields:
   - **`_debug_provider_endpoint`**: Set to your AI provider's chat completions URL.
     - OpenAI: `https://api.openai.com/v1/chat/completions`
     - Anthropic: `https://api.anthropic.com/v1/messages`
   - **`_debug_model_name`**: Set to your preferred model.
     - OpenAI examples: `gpt-4o`, `gpt-4o-mini`
     - Anthropic examples: `claude-opus-4-5`, `claude-sonnet-4-5`
   - **`_debug_max_tokens`**: Recommended `2000` for single spec, `3000` for portfolio review.
   - **`_debug_temperature`**: Recommended `0.2` for consistent diagnostic output.
   - **`_debug_retry_max_attempts`**: Recommended `2`.

4. In the **AI: Analyze Spec Ambiguity** node and **AI: Portfolio Review** node, select your saved credential from the **Credential** dropdown.

5. Save the workflow again after making these changes.

---

## Step 4 — Activate the Workflow

1. In the workflow editor, toggle the workflow to **Active** using the switch in the top-right corner.
2. Click the **Webhook** node to reveal the webhook URL. Copy the **Production URL** (not the test URL).

The webhook URL will look like:
```
https://your-n8n-instance.com/webhook/spec-ambiguity-detector
```

---

## Step 5 — Send a Sample Payload

Use the following example payload to test your setup. Copy it to a file named `test-payload.json`:

```json
{
  "user_id": "pm-test-user",
  "analysis_mode": "single_spec",
  "preferred_tone": "direct",
  "specs": [
    {
      "title": "Smart Notification Center — v1 PRD",
      "type": "prd",
      "sections": ["overview", "goals", "requirements"],
      "raw_text": "The notification center will surface relevant alerts to users in a fast and intuitive way. The backend team will handle delivery. Edge cases will be supported. Notifications must load quickly and feel seamless across devices. The system should be robust enough to handle high volumes without degrading the experience. We will support all major platforms. The notification UI should be clean and unobtrusive."
    }
  ]
}
```

Send the request:

```bash
curl -X POST \
  https://your-n8n-instance.com/webhook/spec-ambiguity-detector \
  -H "Content-Type: application/json" \
  -d @test-payload.json
```

---

## Step 6 — Validate the Response

The response should be a JSON object. Check for these top-level fields:

- `ambiguity_summary` — non-empty string
- `ambiguity_categories` — array containing at least one category
- `severity_level` — one of: `none`, `low`, `medium`, `high`, `critical`
- `vague_terms_detected` — array (should contain "fast", "intuitive", "seamless", "robust" for the test payload)
- `recommended_clarifications` — array of specific, actionable items
- `confidence_band` — one of: `low`, `medium`, `high`
- `needs_human_review` — boolean

Validate the response against the full schema at `schemas/spec-analysis-output-schema.json`.

**If you see an empty response or HTTP 500**: Check the n8n execution log for the failed run. The most common causes are credential misconfiguration, incorrect provider endpoint, or a model name that the provider does not recognize.

---

## Common Setup Mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| Using the test webhook URL instead of the production URL | Workflow does not trigger when sent from external callers | Copy the Production URL from the Webhook node, not the Test URL |
| Credential not assigned to the AI HTTP Request nodes | HTTP 401 from AI provider | Open each AI node and assign the saved credential |
| Wrong `_debug_provider_endpoint` format | HTTP 404 or connection error | Confirm the URL matches the provider's exact API path |
| Model name not available on your account | HTTP 404 or 403 from provider | Verify the model name is available on your API tier |
| Sending `analysis_mode: "portfolio_review"` with only one spec | Thin portfolio output | Portfolio review is designed for 3 or more specs |
| Forgetting to activate the workflow | Webhook returns 404 | Toggle the workflow to Active before sending requests |
| Payload missing the `specs` array | Workflow errors at Normalize node | Ensure your payload matches the structure in `schemas/spec-input-payload-example.json` |

---

## Next Steps

- Read [`docs/credential-setup.md`](credential-setup.md) for security best practices around API key handling
- Read [`docs/customization-guide.md`](customization-guide.md) to adapt the vague term vocabulary, categories, and thresholds to your team's needs
- Run the full test suite from [`docs/testing-guide.md`](testing-guide.md) to confirm end-to-end behavior
