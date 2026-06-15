# Architecture Overview

## Spec Ambiguity Detector — Workflow Architecture

This document describes the internal architecture of the `spec-ambiguity-detector.json` n8n workflow. It explains what each node is responsible for, where deterministic logic runs, where AI is invoked, and how the two analysis modes (single spec and portfolio review) interact.

---

## High-Level Flow

```
Webhook
  └─▶ Provider Config
        └─▶ Normalize Spec Input
              └─▶ Extract Structural Signals
                    └─▶ Detect Deterministic Ambiguity Flags
                          └─▶ Validate Signal Strength
                                ├─▶ [Low Signal] → Low Signal Response → Respond to Webhook
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

## Stage-by-Stage Description

### 1. Webhook

**Type**: n8n Webhook node  
**Method**: POST  
**Purpose**: Entry point for the workflow. Accepts a structured JSON payload from any caller — a CI/CD pipeline, a Slack slash command, a Notion webhook, a manual cURL, or a scheduled trigger.

**Accepts**:
- `user_id` — caller identifier for routing or audit purposes
- `analysis_mode` — either `single_spec` or `portfolio_review`
- `preferred_tone` — `direct`, `explanatory`, or `minimal`
- `specs` — array of one or more spec objects, each containing `title`, `type`, `sections`, and `raw_text`

**Does not**:
- Parse files
- Store payloads
- Authenticate the caller (authentication is handled at the infrastructure level)

---

### 2. Provider Config

**Type**: n8n Set node  
**Purpose**: Centralizes all AI provider configuration into one place. No other node hardcodes the provider endpoint, model name, or retry settings.

**Sets**:
- `_debug_provider_endpoint` — the AI provider base URL (e.g., OpenAI, Anthropic, or self-hosted)
- `_debug_model_name` — the model to use for analysis (e.g., `gpt-4o`, `claude-opus-4-5`)
- `_debug_max_tokens` — output token limit
- `_debug_temperature` — generation temperature (recommended: 0.1–0.3 for diagnostic tasks)
- `_debug_retry_max_attempts` — number of retries on AI call failure

**Note**: All fields use the `_debug_` prefix because they are internal workflow configuration values. They are stripped before the final response is returned to the caller.

---

### 3. Normalize Spec Input

**Type**: n8n Code node (JavaScript)  
**Purpose**: Transforms the raw webhook payload into a normalized internal representation that downstream nodes can rely on regardless of input variation.

**Operations**:
- Extracts and trims `raw_text` from each spec in the `specs` array
- Counts word count and estimated section count per spec
- Detects whether expected structural sections are present (`goals`, `scope`, `assumptions`, `requirements`, `edge_cases`, `dependencies`, `metrics`, `open_questions`, `out_of_scope`)
- Tags each spec with `_debug_normalized: true` to confirm normalization occurred
- Passes through `analysis_mode`, `preferred_tone`, and `user_id` unchanged

**Why this exists**: Input payloads in real use are inconsistent. Normalization isolates downstream logic from input shape variation.

---

### 4. Extract Structural Signals

**Type**: n8n Code node (JavaScript)  
**Purpose**: Scans the normalized text for structural signals that inform ambiguity detection without requiring AI.

**Extracts**:
- Whether the document has an explicit out-of-scope section
- Whether the document has an open questions section
- Whether the document has a named assumptions section
- Whether any section contains ownership language (e.g., "will be owned by," "responsible for," "assigned to")
- Whether any acceptance criteria or success metrics are stated
- Whether the document references external dependencies explicitly (by name, API, or service)
- Estimated reading complexity proxy (average sentence length, paragraph density)

**Output fields** (all prefixed with `_debug_`):
- `_debug_has_out_of_scope_section`
- `_debug_has_open_questions_section`
- `_debug_has_assumptions_section`
- `_debug_has_ownership_signals`
- `_debug_has_acceptance_criteria_signals`
- `_debug_has_named_dependencies`
- `_debug_estimated_word_count`
- `_debug_structural_section_list`

---

### 5. Detect Deterministic Ambiguity Flags

**Type**: n8n Code node (JavaScript)  
**Purpose**: Runs rule-based, non-AI ambiguity detection against the normalized text. This is the first analysis layer. It operates entirely without AI and catches high-confidence signal patterns.

**Deterministic checks performed**:

| Check | Logic |
|---|---|
| Vague term scan | Matches against a configurable vocabulary list including: "fast," "slow," "easy," "hard," "intuitive," "robust," "appropriate," "seamless," "support," "handle," "manage," "user-friendly," "efficient," "scalable," "reliable," "better," "simple," "improve" |
| Missing out-of-scope section | Flags if `_debug_has_out_of_scope_section` is false and document is above the low-signal threshold |
| Missing open questions section | Flags if `_debug_has_open_questions_section` is false |
| Missing assumptions section | Flags if `_debug_has_assumptions_section` is false |
| Ownership signal absence | Flags if `_debug_has_ownership_signals` is false |
| Acceptance criteria absence | Flags if `_debug_has_acceptance_criteria_signals` is false |
| Dependency name absence | Flags if no named external systems, APIs, or services appear in the text |
| Contradictory scope keywords | Looks for co-occurrence of "in scope" / "out of scope" / "not included" with contradicting "will include" / "will support" statements |
| Short spec detection | Word count below 150 triggers `low_signal_noise` pre-flag |

**Output**:
- `_debug_deterministic_flags` — array of flag objects with `flag_type`, `matched_terms`, and `confidence`
- `_debug_vague_terms_found` — list of matched vague terms
- `_debug_pre_severity_estimate` — preliminary severity estimate based on flag count alone

---

### 6. Validate Signal Strength

**Type**: n8n Switch node  
**Purpose**: Decides whether the document has sufficient signal for AI analysis or should be short-circuited as `low_signal_noise`.

**Routing logic**:
- If `_debug_estimated_word_count < 150` OR `_debug_pre_severity_estimate === 'low_signal_noise'` → route to **Low Signal Response**
- Otherwise → route to **Prep Spec Analysis Request**

**Why this matters**: Short-circuiting low-signal specs avoids wasting AI tokens on documents that are too thin to yield useful diagnostics. The caller receives an immediate, honest response explaining why full analysis was not performed.

---

### 7. Low Signal Response

**Type**: n8n Set node  
**Purpose**: Constructs a complete, well-formed diagnostic response for documents that do not have sufficient content for full analysis. No AI is called.

**Response includes**:
- `ambiguity_summary` explaining why the document was classified as low-signal
- `ambiguity_categories: ["low_signal_noise"]`
- `severity_level: "low"` (the document may be low risk precisely because it is too early in drafting)
- `needs_human_review: true`
- `review_reason` describing what additional content is needed for a full diagnostic

---

### 8. Prep Spec Analysis Request

**Type**: n8n Set node  
**Purpose**: Constructs the full prompt payload for the AI analysis call. Merges the system prompt from `prompts/system-spec-analysis.md` with the runtime-interpolated user prompt from `prompts/user-spec-analysis-template.md`.

**Interpolation variables populated**:
- `{{analysis_mode}}` from workflow state
- `{{document_title}}` from normalized spec
- `{{document_type}}` from spec input (e.g., `prd`, `technical_spec`, `user_stories`)
- `{{source_summary}}` — a one-sentence description of what sections were detected
- `{{word_count}}` from structural signal extraction
- `{{section_count}}` from structural signal extraction
- `{{preferred_tone}}` from webhook input
- `{{document_text}}` — the normalized `raw_text`
- `{{known_sections}}` — the detected section list
- `{{detected_open_questions}}` — any open question text detected deterministically
- `{{detected_out_of_scope_items}}` — any out-of-scope text detected deterministically

---

### 9. AI: Analyze Spec Ambiguity

**Type**: n8n HTTP Request node (targeting AI provider)  
**Purpose**: Sends the prepared prompt to the configured AI provider and receives a structured JSON diagnostic response.

**Configuration**:
- Uses credentials from n8n credential store (HTTP Header Auth)
- Applies retry settings from Provider Config node
- Sets `response_format` or equivalent to encourage structured JSON output (provider-dependent)
- Temperature: 0.1–0.3 (low, for consistency)
- Timeout: 60 seconds

**Supported response shapes**:
- OpenAI-style: `choices[0].message.content`
- Anthropic-style: `content[0].text`

Both shapes are handled in the subsequent Parse node.

---

### 10. Parse Spec Analysis

**Type**: n8n Code node (JavaScript)  
**Purpose**: Extracts and validates the AI response. Handles malformed output gracefully.

**Operations**:
- Detects response shape (OpenAI vs. Anthropic) and extracts the content string
- Attempts `JSON.parse()` on the content
- If parsing fails: sets `_debug_parse_failed: true`, constructs a fallback diagnostic with `needs_human_review: true` and `review_reason: "Model output could not be parsed as valid JSON"`
- If parsing succeeds: validates that required top-level fields are present
- Missing required fields trigger partial fallback with human review flag

---

### 11. Deterministic Quality Checks

**Type**: n8n Code node (JavaScript)  
**Purpose**: Post-processes the parsed AI output with deterministic rules. This layer catches cases where the AI output is technically valid JSON but contains quality problems.

**Checks performed**:
- Verifies that every term in `vague_terms_detected` was also in `_debug_vague_terms_found` (prevents AI hallucinating false positives)
- If `severity_level` is `critical`, forces `needs_human_review: true`
- Cross-checks that `recommended_clarifications` count is non-zero when `ambiguity_categories` is non-empty
- Verifies that `confidence_band` is one of the three allowed values (`low`, `medium`, `high`)
- If `contradictory_requirement` is in `ambiguity_categories`, verifies `contradictions_detected` is non-empty
- Adjusts `severity_level` upward if deterministic flags indicate higher severity than AI estimated

---

### 12. Human Review Override

**Type**: n8n Switch node  
**Purpose**: Evaluates the final analysis state and routes accordingly.

**Routes**:
- If `analysis_mode === "portfolio_review"` → **Prep Portfolio Review**
- Otherwise → **Final Response**

Also sets `needs_human_review: true` in any of these conditions:
- `severity_level === "critical"`
- `_debug_parse_failed === true`
- `confidence_band === "low"`
- Deterministic flags count exceeds AI-identified category count by more than 3
- Document word count is below 300 (borderline signal)

---

### 13. Prep Portfolio Review

**Type**: n8n Set node  
**Purpose**: When `analysis_mode` is `portfolio_review`, this node collects the individual spec analyses and constructs the portfolio-level prompt.

**Behavior**:
- Aggregates all spec diagnostic outputs from the `specs` array
- Identifies which specs had `needs_human_review: true`
- Constructs a prompt using `prompts/system-portfolio-review.md` and `prompts/user-portfolio-review-template.md`
- Passes the full set of individual analyses as structured context to the portfolio review AI call

---

### 14. AI: Portfolio Review

**Type**: n8n HTTP Request node  
**Purpose**: Sends the portfolio-level prompt to the AI provider to identify cross-document patterns.

**Same configuration as the single-spec AI call**. The system prompt instructs the model to identify recurring patterns across the spec set rather than analyzing any single document in isolation.

---

### 15. Parse Portfolio Review

**Type**: n8n Code node (JavaScript)  
**Purpose**: Extracts and validates the portfolio AI response. Same fallback logic as the single-spec parse node.

---

### 16. Final Response

**Type**: n8n Code node (JavaScript)  
**Purpose**: Constructs the clean, public-facing response by stripping all `_debug_` prefixed fields from the workflow state.

**Operations**:
- Iterates over all keys in the output object
- Removes any key beginning with `_debug_`
- Structures the final JSON response from the validated, cleaned fields
- Adds `generated_at` timestamp (ISO 8601)
- Adds `analysis_mode` for caller context

---

### 17. Respond to Webhook

**Type**: n8n Respond to Webhook node  
**Purpose**: Returns the final cleaned response to the caller as HTTP 200 with `Content-Type: application/json`.

On error paths, returns HTTP 422 with a structured error object explaining what went wrong.

---

## Debug Field Convention

All internal workflow fields use the `_debug_` prefix:

```
_debug_provider_endpoint
_debug_model_name
_debug_normalized
_debug_deterministic_flags
_debug_vague_terms_found
_debug_pre_severity_estimate
_debug_estimated_word_count
_debug_has_out_of_scope_section
_debug_parse_failed
...
```

The **Final Response** node removes all of these before returning the response to the caller. This means:
- Callers never see internal processing artifacts
- Debug information is available during workflow development via n8n's execution inspector
- Changing internal field names never breaks the public API contract

---

## Where AI Is Used

| Stage | AI Used? | Purpose |
|---|---|---|
| Normalize Spec Input | No | Rule-based normalization |
| Extract Structural Signals | No | Pattern matching and structural detection |
| Detect Deterministic Ambiguity Flags | No | Vocabulary scan and structural checks |
| Validate Signal Strength | No | Threshold-based routing |
| Low Signal Response | No | Immediate structured fallback |
| AI: Analyze Spec Ambiguity | **Yes** | Deep ambiguity pattern classification |
| Deterministic Quality Checks | No | Post-processing and hallucination reduction |
| AI: Portfolio Review | **Yes** | Cross-document pattern analysis |
| Final Response | No | Field stripping and formatting |

---

## Portfolio Review Mode

When `analysis_mode` is `portfolio_review`, the workflow:

1. Processes each spec through the single-spec analysis pipeline individually
2. Collects all individual diagnostic outputs
3. Passes the full set to the portfolio review AI call
4. Returns a single portfolio-level response containing:
   - Cross-document pattern analysis
   - Recurring ambiguity category identification
   - Top-risk spec identification
   - Portfolio-level recommendations

This mode is designed for weekly reviews, sprint planning, or backlog quality audits where multiple specs need to be evaluated in relation to each other.

---

## Low-Signal Short-Circuiting

Documents below 150 words are short-circuited before any AI call is made. This is intentional:

- A 50-word spec cannot be meaningfully analyzed for ownership gaps or acceptance criteria patterns
- Calling AI on low-signal input wastes tokens and produces low-confidence, often misleading output
- The `low_signal_noise` classification is itself a useful diagnostic — it tells the author that the document is not yet ready for structured review

The threshold (150 words) is configurable in the `Validate Signal Strength` node.

---

## How Contradictions Are Handled

Contradictory requirements are detected both deterministically (keyword co-occurrence in scope-related sections) and by AI (semantic contradiction detection). When both layers flag contradictions:

- `contradictory_requirement` is added to `ambiguity_categories`
- `contradictions_detected` is populated with specific statement pairs
- `severity_level` is escalated to at least `high`
- `needs_human_review` is set to `true`
- `recommended_clarifications` includes specific resolution steps for each contradiction
