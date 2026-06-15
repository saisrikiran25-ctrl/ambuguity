# Customization Guide

## Spec Ambiguity Detector — Adapting the Workflow to Your Team

This guide explains every customizable aspect of the Spec Ambiguity Detector, where to make each change, and what to watch out for when doing so.

---

## Where Prompt Text Lives

This is the most important thing to understand before customizing:

**Prompt text exists in two places:**

1. The `.md` files in the `prompts/` directory — these are the canonical, human-readable versions you should edit first
2. Inside the workflow nodes themselves — specifically the **Prep Spec Analysis Request** and **Prep Portfolio Review** nodes, which embed the prompt text as JavaScript template literals

**When you edit a prompt, you must update both:**
- Edit the `.md` file first to get the wording right
- Then copy the updated text into the corresponding workflow node

If you only edit the `.md` file, the workflow will continue using the old prompt. If you only edit the workflow node, your `.md` file will be out of sync and misleading to future maintainers.

The `.md` files are the source of truth. The workflow nodes contain the operational copy.

---

## Customizing the Vague Term Vocabulary

**Location**: `Detect Deterministic Ambiguity Flags` node (Code node in the workflow)

The default vocabulary list includes these terms:

```
fast, slow, easy, hard, difficult, simple, complex, intuitive, robust, 
appropriate, seamless, support, handle, manage, user-friendly, efficient, 
scalable, reliable, better, improve, enhance, optimal, sufficient, 
adequate, reasonable, significant, minimal, large, small, good, bad
```

**To add terms**:
1. Open the `Detect Deterministic Ambiguity Flags` node
2. Find the `VAGUE_TERMS` array at the top of the code block
3. Add your terms to the array
4. Save the workflow

**To remove terms**:
Remove the term from the array. Be aware that removing common terms will reduce deterministic detection coverage.

**Team-specific vocabulary**: If your team has domain-specific vague terms (e.g., "production-ready," "enterprise-grade," "platform-native"), add them here. The deterministic scan is case-insensitive and matches whole words and common inflections.

**Note**: Vague terms detected deterministically are stored in `_debug_vague_terms_found`. These are cross-referenced against AI output in the `Deterministic Quality Checks` node to reduce false positives.

---

## Customizing Ambiguity Categories

**Location**: System prompt files + workflow schema validation code

The ten canonical categories are:

```
vague_language
missing_acceptance_criteria
undefined_ownership
missing_decision_point
scope_boundary_gap
dependency_blind_spot
untestable_requirement
contradictory_requirement
assumption_without_validation
low_signal_noise
```

**To add a new category**:
1. Add the category name to the enum in `schemas/spec-analysis-output-schema.json` under `ambiguity_categories.items.enum`
2. Add a description of the category to `prompts/system-spec-analysis.md` in the Allowed Ambiguity Categories section
3. Add the same description to `prompts/system-portfolio-review.md`
4. Update the workflow's prompt prep nodes to reflect the new category in the output contract
5. Update the `Deterministic Quality Checks` node if any deterministic logic should apply to the new category
6. Update this customization guide

**Do not rename core categories** without updating all of the above locations simultaneously. Category names are used as enum values in schemas, as string constants in workflow code, and as terms in prompts. A rename in one place creates silent inconsistency.

---

## Customizing Severity Thresholds

**Location**: `Deterministic Quality Checks` node (Code node in the workflow)

The severity escalation logic works as follows by default:

| Condition | Effect |
|---|---|
| 0 ambiguity categories detected | `severity_level: "none"` |
| 1–2 categories, none are high-risk | `severity_level: "low"` |
| 3–4 categories, or any ownership/decision gap | `severity_level: "medium"` |
| 5+ categories, or critical ownership gaps | `severity_level: "high"` |
| `contradictory_requirement` present | Escalate to at least `high` |
| Multiple critical flags plus AI severity is high | `severity_level: "critical"` |

**To adjust thresholds**:
1. Open the `Deterministic Quality Checks` node
2. Find the `SEVERITY_RULES` object
3. Modify the category count thresholds or the escalation conditions
4. Save the workflow

**Example**: If your team treats any missing ownership flag as immediately `high`, add that as an escalation rule.

---

## Customizing the Tone

**Location**: `preferred_tone` field in the webhook input payload

The `preferred_tone` field in the input payload controls how the AI frames its output. Three values are supported:

| Tone | Behavior |
|---|---|
| `direct` | Short, declarative sentences. No hedging. Recommendations are stated as actions. |
| `explanatory` | Slightly longer output with brief rationale for each recommendation. Useful when sharing with authors who want context. |
| `minimal` | Compressed output — summary, categories, and clarifications only. No extended descriptions. |

**To change the default**: The `Normalize Spec Input` node sets a default tone of `direct` if none is provided. To change the default, modify the fallback assignment in that node.

**To add a new tone value**: Update the system prompt to define the behavior for the new tone, and add it to the `preferred_tone` enum in `schemas/spec-input-payload-example.json`.

---

## Customizing Section Expectations by Document Type

**Location**: `Extract Structural Signals` node (Code node in the workflow)

The workflow applies different section-presence expectations depending on the `type` field of each spec:

| Document Type | Required Sections Expected |
|---|---|
| `prd` | goals, scope, requirements, assumptions, open_questions, out_of_scope, metrics |
| `technical_spec` | overview, requirements, interfaces, dependencies, edge_cases, testing |
| `user_stories` | user goals, acceptance_criteria, edge_cases |
| `implementation_brief` | scope, approach, dependencies, open_questions |
| `feature_request` | problem, proposed_solution, success_metrics |

**To adjust these expectations**:
1. Open the `Extract Structural Signals` node
2. Find the `SECTION_EXPECTATIONS` object
3. Add, remove, or modify the expected section lists per document type
4. Save the workflow

Missing expected sections are flagged in `_debug_deterministic_flags` and included in the AI prompt as context signals.

---

## Customizing Portfolio Review Behavior

**Location**: `Prep Portfolio Review` node + `prompts/system-portfolio-review.md`

**Minimum spec count**: By default, the portfolio review requires at least 3 specs to produce meaningful cross-document patterns. Below that count, the output is still returned but `confidence_band` is set to `low` and `needs_human_review` is set to `true`.

To change the minimum:
1. Open the `Prep Portfolio Review` node
2. Find the `MIN_PORTFOLIO_SPECS` constant
3. Change the value
4. Save the workflow

**Pattern prioritization**: The portfolio review prompt currently prioritizes these cross-doc patterns:
- Recurring ownership ambiguity
- Repeated missing acceptance criteria
- Weak scope discipline
- Common missing decisions
- Specs at highest delivery risk

To change the priority or add new cross-doc patterns, edit `prompts/system-portfolio-review.md` and update the corresponding workflow node.

---

## Customizing the AI Provider

**Location**: `Provider Config` node

The workflow supports any AI provider with an HTTP API accepting JSON request bodies. The two officially supported shapes are:

**OpenAI-compatible** (e.g., OpenAI, Azure OpenAI, Groq, Mistral):
```json
{
  "model": "gpt-4o",
  "messages": [
    {"role": "system", "content": "..."},
    {"role": "user", "content": "..."}
  ],
  "temperature": 0.2,
  "max_tokens": 2000
}
```

**Anthropic-compatible**:
```json
{
  "model": "claude-opus-4-5",
  "system": "...",
  "messages": [
    {"role": "user", "content": "..."}
  ],
  "temperature": 0.2,
  "max_tokens": 2000
}
```

The `Parse Spec Analysis` and `Parse Portfolio Review` nodes detect which response shape was returned and extract the content string accordingly.

**To switch providers**:
1. Update `_debug_provider_endpoint` in the Provider Config node
2. Update `_debug_model_name` to a model available at that endpoint
3. Update your credential in n8n to match the new provider's auth format
4. Test with the full suite in [`docs/testing-guide.md`](testing-guide.md)

---

## Customizing the Low-Signal Threshold

**Location**: `Validate Signal Strength` node

By default, documents under 150 words are classified as `low_signal_noise` and short-circuited before AI invocation.

To change this threshold:
1. Open the `Validate Signal Strength` node
2. Find the `LOW_SIGNAL_WORD_COUNT_THRESHOLD` constant
3. Adjust the value
4. Save the workflow

**Raising the threshold** (e.g., to 300) means more documents get short-circuited without AI analysis. Use this if you frequently receive thin draft specs and want to enforce a minimum quality gate before AI review.

**Lowering the threshold** (e.g., to 75) allows shorter documents through to AI analysis. This may produce lower-quality diagnostics for very short specs.

---

## Customizing the Human Review Flag Conditions

**Location**: `Human Review Override` node

By default, `needs_human_review` is set to `true` when any of the following conditions are met:
- `severity_level === "critical"`
- `_debug_parse_failed === true`
- `confidence_band === "low"`
- Deterministic flag count exceeds AI category count by more than 3
- Document word count is under 300

To add or change these conditions, edit the switch logic in the `Human Review Override` node.

---

## Prompt Customization Summary Table

| What to Customize | File | Workflow Node |
|---|---|---|
| System-level analysis instructions | `prompts/system-spec-analysis.md` | Prep Spec Analysis Request |
| Runtime analysis template | `prompts/user-spec-analysis-template.md` | Prep Spec Analysis Request |
| System-level portfolio instructions | `prompts/system-portfolio-review.md` | Prep Portfolio Review |
| Runtime portfolio template | `prompts/user-portfolio-review-template.md` | Prep Portfolio Review |
| Vague term vocabulary | N/A (workflow only) | Detect Deterministic Ambiguity Flags |
| Section expectations by doc type | N/A (workflow only) | Extract Structural Signals |
| Severity escalation rules | N/A (workflow only) | Deterministic Quality Checks |
| Low-signal threshold | N/A (workflow only) | Validate Signal Strength |
| Human review conditions | N/A (workflow only) | Human Review Override |
