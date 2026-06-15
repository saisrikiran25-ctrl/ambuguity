# Spec Ambiguity Detector Automation

> **Turns vague specs into executable clarity by detecting ambiguity before development starts.**

An AI-powered n8n workflow that reviews PRDs, feature specs, implementation briefs, and requirement documents to identify ambiguity patterns before engineering begins. Returns a structured diagnostic — not a rewrite, not a summary — so your team can resolve gaps before they become delivery risk.

---

## What Is Spec Ambiguity?

Spec ambiguity is any condition in a requirements document that allows two or more reasonable interpretations. It is not a style problem. It is a delivery risk.

When a spec says "the system should respond quickly," different engineers will ship different things. When a spec says "the notification system will handle edge cases," no one knows who owns it or what the edge cases are. When requirements describe behavior without acceptance criteria, QA writes tests against assumptions.

Requirements quality guidance consistently identifies the same root causes:
- Subjective or unquantified language
- Missing or implicit decisions
- Undefined ownership and responsibility boundaries
- Incomplete scope definitions
- Untestable acceptance criteria
- Hidden dependencies and external constraints
- Assumptions treated as established facts

This product detects those conditions early, before tickets are broken down, before engineers implement assumptions, and before scope quietly expands.

---

## What This Is / What This Is Not

| This is | This is not |
|---|---|
| A requirements-quality diagnostic tool | A spec writer or content generator |
| An ambiguity pattern classifier | A summarizer or document compressor |
| A structured diagnostic with actionable recommendations | A project management or ticketing tool |
| A pre-build audit automation | A legal or compliance reviewer |
| A workflow you can route into Slack, Notion, Linear, or Jira | A replacement for product judgment |

---

## Ambiguity Pattern Types

The detector classifies ambiguity into ten canonical categories:

| Pattern Type | What It Looks Like |
|---|---|
| `vague_language` | Terms like "fast," "intuitive," "robust," "seamless," "appropriate," or "support" without measurable criteria. |
| `missing_acceptance_criteria` | Requirements that describe desired behavior but provide no verifiable success conditions. |
| `undefined_ownership` | Statements about what "the system" or "the team" will do, without specifying which role, team, or service is responsible. |
| `missing_decision_point` | A feature direction described without documenting the unresolved choices, tradeoffs, or product decisions still open. |
| `scope_boundary_gap` | Missing definition of what is explicitly excluded, deferred, or out of scope for this release. |
| `dependency_blind_spot` | Quiet reliance on external APIs, services, design assets, legal review, or stakeholder input without stating those dependencies. |
| `untestable_requirement` | Requirements written in terms of intent rather than observable, verifiable behavior. |
| `contradictory_requirement` | Two or more statements in the document that cannot both be true or that conflict on scope, behavior, or constraints. |
| `assumption_without_validation` | Statements presented as facts that are actually unvalidated premises — market assumptions, user behaviors, or system capabilities that have not been confirmed. |
| `low_signal_noise` | A document too short, too abstract, or too thin to yield meaningful diagnostic signals. Analysis cannot proceed reliably. |

---

## Who It Is For

- **Product managers** who want to catch weak requirements before engineering handoff
- **Engineers** who want clearer implementation inputs and fewer assumption-driven builds
- **Designers** who need missing flow logic, edge cases, and interaction assumptions surfaced early
- **QA and test engineers** who need testable acceptance criteria rather than broad intent statements
- **Founders and small teams** who move fast and skip formal review, but still need decision clarity

---

## Key Features

- **Dual analysis modes**: `single_spec` for individual documents and `portfolio_review` for cross-document pattern analysis
- **Ten canonical ambiguity categories** with structured severity classification
- **Deterministic pre-checks** before AI invocation — vague term scanning, section presence validation, ownership signal detection
- **Low-signal short-circuit**: specs too thin to analyze are flagged immediately without wasting AI calls
- **Structured JSON output** designed for routing into downstream tools (Slack, Notion, Linear, Jira, webhooks)
- **Human review flags** for edge cases — malformed output, contradictory signals, critical severity
- **Portfolio-level pattern detection** — recurring ownership gaps, repeated missing acceptance criteria, highest-risk specs across a batch
- **Debug field stripping** — internal `_debug_` fields are never exposed in the final response
- **No hardcoded secrets** — all credentials are wired through n8n credential store

---

## Workflow Overview

```
Webhook
  └─▶ Provider Config
        └─▶ Normalize Spec Input
              └─▶ Extract Structural Signals
                    └─▶ Detect Deterministic Ambiguity Flags
                          └─▶ Validate Signal Strength
                                ├─▶ [Low Signal] → Low Signal Response
                                │                       └─▶ Respond to Webhook
                                └─▶ [Valid] → Prep Spec Analysis Request
                                                  └─▶ AI: Analyze Spec Ambiguity
                                                        └─▶ Parse Spec Analysis
                                                              └─▶ Deterministic Quality Checks
                                                                    └─▶ Human Review Override
                                                                          ├─▶ [portfolio_review] → Prep Portfolio Review
                                                                          │                           └─▶ AI: Portfolio Review
                                                                          │                                 └─▶ Parse Portfolio Review
                                                                          └─▶ Final Response
                                                                                └─▶ Respond to Webhook
```

---

## Directory Structure

```
spec-ambiguity-detector-automation/
├── README.md                               ← This file
├── assets/
│   └── architecture-overview.md           ← Node-by-node workflow architecture
├── docs/
│   ├── setup-guide.md                     ← Import, configure, and first run
│   ├── credential-setup.md                ← Credential wiring without storing secrets
│   ├── customization-guide.md             ← Adapting prompts, thresholds, categories
│   └── testing-guide.md                   ← 7+ test scenarios with expected behavior
├── prompts/
│   ├── system-spec-analysis.md            ← Master system prompt for single-spec analysis
│   ├── user-spec-analysis-template.md     ← Runtime interpolation template for single spec
│   ├── system-portfolio-review.md         ← System prompt for multi-spec portfolio review
│   └── user-portfolio-review-template.md  ← Runtime template for portfolio review
├── schemas/
│   ├── spec-input-payload-example.json    ← Realistic example webhook input payload
│   ├── spec-analysis-output-schema.json   ← JSON Schema (draft-07) for single spec output
│   └── portfolio-review-output-schema.json← JSON Schema (draft-07) for portfolio output
└── workflow/
    └── spec-ambiguity-detector.json       ← Importable n8n workflow
```

---

## Setup Summary

1. Import `workflow/spec-ambiguity-detector.json` into your n8n instance via **Workflows → Import from File**
2. Create an **HTTP Header Auth** credential in n8n for your AI provider (OpenAI or Anthropic)
3. Update the **Provider Config** node with your model name and endpoint
4. Send a POST request to the workflow webhook URL with a payload matching `schemas/spec-input-payload-example.json`
5. Validate the response against `schemas/spec-analysis-output-schema.json`

See [`docs/setup-guide.md`](docs/setup-guide.md) for the full walkthrough.

---

## Credential Note

No API keys or secrets are included in this repository. All credentials must be configured in your own n8n instance through the built-in credential store. The workflow JSON contains no hardcoded secrets. See [`docs/credential-setup.md`](docs/credential-setup.md).

---

## Example Input

```json
{
  "user_id": "pm-team-atlas",
  "analysis_mode": "single_spec",
  "preferred_tone": "direct",
  "specs": [
    {
      "title": "Smart Notification Center — v1 PRD",
      "type": "prd",
      "sections": ["overview", "goals", "requirements", "edge_cases"],
      "raw_text": "The notification center will surface relevant alerts to users in a fast and intuitive way. The backend team will handle delivery. Edge cases will be supported. Notifications must load quickly and feel seamless across devices. The system should be robust enough to handle high volumes without degrading the experience."
    }
  ]
}
```

---

## Example Output

```json
{
  "ambiguity_summary": "This document contains five distinct ambiguity patterns across vague language, missing acceptance criteria, undefined ownership, and scope boundary gaps. The terms 'fast', 'intuitive', 'seamless', and 'robust' appear without measurable criteria. Ownership of notification delivery, edge case handling, and retry behavior is not assigned. No acceptance criteria are defined for any of the stated requirements. The document does not specify which edge cases are in or out of scope.",
  "ambiguity_categories": [
    "vague_language",
    "missing_acceptance_criteria",
    "undefined_ownership",
    "scope_boundary_gap"
  ],
  "severity_level": "high",
  "highest_risk_sections": ["requirements", "edge_cases"],
  "vague_terms_detected": ["fast", "intuitive", "seamless", "robust", "high volumes"],
  "missing_decisions": [
    "No decision recorded on notification delivery mechanism (push, email, in-app, or combination)",
    "No decision on retry behavior or failure handling",
    "No decision on read/unread state persistence"
  ],
  "undefined_ownership_flags": [
    "'The backend team will handle delivery' — does not specify service, squad, or individual owner",
    "'Edge cases will be supported' — no owner assigned for edge case definition or implementation"
  ],
  "non_testable_requirements": [
    "'Notifications must load quickly' — no latency threshold defined",
    "'Feel seamless across devices' — no device matrix or breakpoint criteria defined",
    "'Robust enough to handle high volumes' — no volume threshold or degradation threshold stated"
  ],
  "dependency_gaps": [
    "Push notification delivery appears to depend on a third-party provider — not named or linked",
    "Device compatibility implies a supported device list — not referenced"
  ],
  "scope_boundary_gaps": [
    "No out-of-scope section or deferred items listed",
    "Edge cases referenced but not enumerated — scope of 'supported edge cases' is undefined"
  ],
  "contradictions_detected": [],
  "assumptions_needing_validation": [
    "Assumes users want a notification center rather than per-feature notification surfaces",
    "Assumes 'high volume' is a likely scenario — no data cited"
  ],
  "recommended_clarifications": [
    "Define the exact latency threshold for 'fast' notification loading (e.g., p95 < 800ms)",
    "Assign explicit ownership for notification delivery infrastructure and retry logic",
    "List the edge cases in scope for v1 and explicitly exclude the remainder",
    "Define the device/platform matrix for 'seamless across devices'",
    "State the volume threshold that constitutes 'high volume' and the acceptable degradation ceiling"
  ],
  "recommended_next_step": "Return this spec to the author with the five clarification items above before engineering breakdown. The document is not implementation-ready in its current state.",
  "confidence_band": "high",
  "needs_human_review": false,
  "review_reason": null
}
```

---

## Example Clarification Recommendations

Recommendations from this tool are always specific and tied to a named ambiguity problem.

| Bad Recommendation | Good Recommendation |
|---|---|
| "Improve clarity throughout." | "Define the exact latency threshold for 'fast search results' (e.g., p95 < 500ms under 10k concurrent users)." |
| "Add acceptance criteria." | "The requirement 'notifications should feel seamless' has no measurable success condition. Add: notification render time < 200ms on standard connection, zero dropped notifications under 1k/min load." |
| "Clarify ownership." | "Assign explicit ownership for notification retry behavior — currently implied to be backend but not confirmed. Confirm: which squad, which service, and who approves the retry policy." |
| "Fix scope." | "The document references 'edge case handling' without listing the edge cases. Either enumerate them or explicitly state that edge case definition is deferred to a separate technical spec." |

---

## Customization Summary

The following aspects of the detector are designed to be customized:

- **Vague term vocabulary** — extend or modify the deterministic term list in the workflow's `Detect Deterministic Ambiguity Flags` node
- **Ambiguity categories** — the canonical ten categories can be extended, but must remain consistent across prompts, schemas, and workflow nodes
- **Severity thresholds** — the conditions for escalating severity can be adjusted in the `Deterministic Quality Checks` node
- **Tone** — the `preferred_tone` field in the input payload controls whether output is `direct`, `explanatory`, or `minimal`
- **Portfolio review behavior** — the minimum number of specs and the cross-doc pattern logic can be tuned in the portfolio review nodes
- **Section expectations by document type** — the workflow treats PRDs, technical specs, and user story sets differently; these rules are configurable
- **Prompt wording** — prompt text exists both in `.md` files (for editing) and inside the workflow nodes (for runtime); changes must be applied in both places

See [`docs/customization-guide.md`](docs/customization-guide.md) for full details.

---

## Testing Summary

The testing guide covers seven scenarios:

1. Happy path PRD with vague language → expects `high` severity, five or more ambiguity categories
2. Spec with no acceptance criteria → expects `missing_acceptance_criteria` as primary category
3. Undefined ownership across key decisions → expects `undefined_ownership` flags across multiple sections
4. Contradictory scope statements → expects `contradictory_requirement` category and `needs_human_review: true`
5. Low-signal short spec → expects short-circuit before AI call, `low_signal_noise` classification
6. Malformed model output → expects graceful fallback and `needs_human_review: true`
7. Portfolio review mode → expects cross-doc pattern summary and top-risk spec identification

See [`docs/testing-guide.md`](docs/testing-guide.md) for full scenario payloads and expected behavior.

---

## Limitations

- **Not a spec validator**: The detector identifies ambiguity patterns; it does not validate factual accuracy or domain correctness.
- **AI output variability**: The AI analysis step can produce slightly different outputs for the same input. The deterministic pre- and post-processing layers reduce this variability but do not eliminate it.
- **Very short documents**: Specs under approximately 150 words will be flagged as `low_signal_noise` and will not receive a full analysis. This is intentional.
- **No document parsing**: The workflow accepts text input, not file uploads. PDFs, Word documents, and Notion exports must be converted to plain text before submission.
- **No memory across runs**: Each invocation is independent. The workflow does not retain history across calls unless you build a persistence layer.
- **Portfolio review requires structured inputs**: The portfolio review mode works best when individual spec analyses have already been run and their outputs are passed as the portfolio payload.
- **Language**: The detector is optimized for English-language documents. Performance on other languages is untested.

---

## Disclaimer

This tool is a diagnostic aid. It surfaces patterns and recommendations based on AI interpretation of text inputs. It can produce false positives (flagging language that is intentionally abstract) and false negatives (missing ambiguity that requires domain expertise to recognize). The output should be used as a starting point for human review, not as a final quality gate. Product judgment cannot be automated.

---

## License

MIT License. See `LICENSE` for details.

---

*Day 20 — Public Automation Series*
