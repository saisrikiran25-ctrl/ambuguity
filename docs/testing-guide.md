# Testing Guide

## Spec Ambiguity Detector — Test Scenarios and Expected Behavior

This guide documents seven test scenarios covering the main behavioral paths of the workflow. Each scenario includes the payload to send, the expected workflow behavior, expected output characteristics, and whether AI should be invoked.

Use these tests to validate your setup after import, after prompt changes, after provider switches, or after any customization.

---

## How to Run Tests

1. Activate the workflow in n8n
2. Copy the Production Webhook URL from the Webhook node
3. Send each test payload using cURL, Postman, or Insomnia
4. Compare the response against the expected behavior described below
5. Use the n8n execution inspector to confirm which nodes were called and what their outputs were

---

## Test 1 — Happy Path: PRD with Vague Language

**Scenario**: A real-looking PRD with multiple vague terms, no acceptance criteria, no ownership, and no out-of-scope section. This is the core happy path.

**Expected workflow path**: Webhook → Provider Config → Normalize → Extract Signals → Deterministic Flags → Validate Signal Strength → [Valid] → Prep Analysis → AI → Parse → Quality Checks → Human Review Override → Final Response → Respond

**AI should be called**: Yes

**Payload**:
```json
{
  "user_id": "test-pm-01",
  "analysis_mode": "single_spec",
  "preferred_tone": "direct",
  "specs": [
    {
      "title": "Smart Notification Center — v1 PRD",
      "type": "prd",
      "sections": ["overview", "goals", "requirements"],
      "raw_text": "The notification center will surface relevant alerts to users in a fast and intuitive way. The backend team will handle delivery. Edge cases will be supported. Notifications must load quickly and feel seamless across devices. The system should be robust enough to handle high volumes without degrading the experience. The UI must be clean, modern, and unobtrusive. Users should find it easy to manage their preferences. The notification experience should feel natural across all platforms we support."
    }
  ]
}
```

**Expected behavior**:
- `severity_level`: `high`
- `ambiguity_categories`: should include at minimum `vague_language`, `missing_acceptance_criteria`, `undefined_ownership`, `scope_boundary_gap`
- `vague_terms_detected`: should include `fast`, `intuitive`, `seamless`, `robust`, `easy`, `clean`
- `undefined_ownership_flags`: non-empty — at minimum "the backend team" is flagged as vague ownership
- `non_testable_requirements`: non-empty — "load quickly", "feel seamless", "robust enough" are all non-testable
- `recommended_clarifications`: at least 4 specific items
- `needs_human_review`: `false` (this is a well-formed response, just a bad spec)
- `confidence_band`: `high`

**Severity tendency**: High. Multiple categories, multiple vague terms, no measurable criteria anywhere.

---

## Test 2 — Missing Acceptance Criteria

**Scenario**: A spec with clearly stated requirements but zero acceptance criteria or success metrics. This isolates the `missing_acceptance_criteria` category.

**AI should be called**: Yes

**Payload**:
```json
{
  "user_id": "test-pm-02",
  "analysis_mode": "single_spec",
  "preferred_tone": "direct",
  "specs": [
    {
      "title": "User Authentication — Password Reset Flow",
      "type": "technical_spec",
      "sections": ["overview", "requirements", "edge_cases"],
      "raw_text": "The password reset flow must allow users to reset their password via email. When a user requests a reset, the system sends an email with a reset link. The link expires after a set period. Users who click an expired link see an error. Users who click a valid link are taken to a form to enter a new password. The new password must meet complexity requirements. After a successful reset, the user is redirected to the login page. Invalid attempts are logged. Suspicious patterns trigger a review flag."
    }
  ]
}
```

**Expected behavior**:
- `ambiguity_categories`: should include `missing_acceptance_criteria` as primary category
- Additional categories: `vague_language` (e.g., "a set period," "complexity requirements," "suspicious patterns"), `undefined_ownership` (who defines "suspicious patterns"?)
- `non_testable_requirements`: "a set period" — no numeric value; "complexity requirements" — not defined; "suspicious patterns" — not defined
- `recommended_clarifications`: should specifically call out missing numeric values (link expiry time, password complexity rules, attempt thresholds)
- `severity_level`: `medium` to `high`
- `needs_human_review`: `false`

**Severity tendency**: Medium. The spec is coherent and describes a real flow, but critical parameters are undefined.

---

## Test 3 — Undefined Ownership Across Key Decisions

**Scenario**: A spec where multiple decisions and responsibilities are described in passive voice or attributed to abstract actors ("the system," "the team," "we"). This isolates the `undefined_ownership` category.

**AI should be called**: Yes

**Payload**:
```json
{
  "user_id": "test-pm-03",
  "analysis_mode": "single_spec",
  "preferred_tone": "direct",
  "specs": [
    {
      "title": "Data Export Feature — v2 Spec",
      "type": "prd",
      "sections": ["overview", "requirements", "dependencies", "open_questions"],
      "raw_text": "The data export feature will allow users to download their account data. The system will support CSV and JSON formats. Exports will be triggered on request. The data pipeline team will ensure data completeness. Legal review will be handled before launch. The notification when an export is ready will be coordinated with the relevant teams. Error handling for failed exports will be addressed. The compliance requirements will need to be finalized. The export format specification will be agreed upon by stakeholders. Monitoring for export job failures will be set up by someone on the infrastructure side."
    }
  ]
}
```

**Expected behavior**:
- `ambiguity_categories`: should include `undefined_ownership` as primary
- `undefined_ownership_flags`: should flag "data pipeline team" (no named squad), "relevant teams" (no named recipients), "someone on the infrastructure side" (not a role), "stakeholders" (not named)
- `missing_decisions`: legal review ownership, compliance requirements, export format specification
- `severity_level`: `high`
- `recommended_clarifications`: should name each unowned decision and ask for the responsible role or team
- `needs_human_review`: `false` unless model confidence is low

**Severity tendency**: High. Multiple unowned decisions that would block engineering.

---

## Test 4 — Contradictory Scope Statements

**Scenario**: A spec that explicitly includes a feature in one section and excludes it in another. This tests `contradictory_requirement` detection and the `needs_human_review` escalation.

**AI should be called**: Yes

**Payload**:
```json
{
  "user_id": "test-pm-04",
  "analysis_mode": "single_spec",
  "preferred_tone": "direct",
  "specs": [
    {
      "title": "Search Feature — Q3 Scope Brief",
      "type": "implementation_brief",
      "sections": ["scope", "requirements", "out_of_scope"],
      "raw_text": "This release includes full-text search across all user-generated content including comments, messages, and uploaded documents. The search will support filters by date, author, and content type. Out of scope for this release: document search. Search will cover only structured content (messages and user profiles). Advanced filtering is not included. The search index will cover all content types the platform supports. Real-time indexing is in scope. Real-time indexing is deferred to a later release."
    }
  ]
}
```

**Expected behavior**:
- `ambiguity_categories`: should include `contradictory_requirement` and `scope_boundary_gap`
- `contradictions_detected`: at minimum two contradictions — "document search in scope" vs. "document search out of scope," "real-time indexing in scope" vs. "real-time indexing deferred"
- `severity_level`: `high` or `critical`
- `needs_human_review`: **`true`** — contradictions require human resolution, not AI recommendation
- `review_reason`: should explain that contradictions cannot be resolved by the detector and require author clarification

**Severity tendency**: High to critical. Contradictory scope in a single document creates immediate delivery risk.

---

## Test 5 — Low-Signal Short Spec

**Scenario**: A document so short and abstract that it cannot yield meaningful analysis. This tests the short-circuit path before AI invocation.

**AI should be called**: **No** — the workflow should detect low signal and return immediately.

**Payload**:
```json
{
  "user_id": "test-pm-05",
  "analysis_mode": "single_spec",
  "preferred_tone": "direct",
  "specs": [
    {
      "title": "New Dashboard Feature",
      "type": "feature_request",
      "sections": ["overview"],
      "raw_text": "We need a better dashboard. It should show the important things and be easy to use. Make it good."
    }
  ]
}
```

**Expected behavior**:
- Workflow routes to `Low Signal Response` node — **no AI call is made**
- `ambiguity_categories`: `["low_signal_noise"]`
- `severity_level`: `low`
- `needs_human_review`: `true`
- `review_reason`: explanation that the document has insufficient content for a full diagnostic — should state what sections and minimum word count are needed
- `recommended_next_step`: should instruct the author to expand the document before requesting analysis
- Response time should be noticeably faster than AI-invoked scenarios (no AI latency)

**To confirm AI was not called**: Check the n8n execution inspector — the `AI: Analyze Spec Ambiguity` node should not appear in the execution path for this test.

---

## Test 6 — Malformed Model Output

**Scenario**: Simulates a situation where the AI provider returns a response that cannot be parsed as valid JSON. This tests the graceful fallback in the `Parse Spec Analysis` node.

**How to simulate this**: Temporarily modify the system prompt to instruct the model to return plain prose instead of JSON. Or, if you have access to n8n's node mocking, inject a malformed string into the Parse node directly.

**AI should be called**: Yes (and its output is intentionally invalid)

**Expected behavior**:
- `Parse Spec Analysis` node sets `_debug_parse_failed: true`
- `needs_human_review`: **`true`**
- `review_reason`: `"Model output could not be parsed as valid JSON. Manual review of the spec is required."`
- `ambiguity_summary`: a fallback string explaining that automated analysis failed
- `confidence_band`: `low`
- `severity_level`: not escalated (since no analysis was completed)
- The workflow does **not crash** — it returns a structured response with an honest error state

**What to verify**: The caller receives an HTTP 200 with a valid JSON body. The workflow does not return an HTTP 500 for malformed model output.

---

## Test 7 — Portfolio Review Mode

**Scenario**: Three specs from different teams are submitted together for portfolio-level pattern analysis. This tests the full portfolio review path.

**AI should be called**: Yes, twice — once for each spec (or in batch, depending on configuration) and once for the portfolio review itself.

**Payload**:
```json
{
  "user_id": "test-pm-07",
  "analysis_mode": "portfolio_review",
  "preferred_tone": "direct",
  "specs": [
    {
      "title": "Mobile Push Notifications — v1",
      "type": "prd",
      "sections": ["overview", "requirements"],
      "raw_text": "Push notifications will be sent to users when important events occur. The backend will handle scheduling. Notifications must be fast and reliable. Users can opt out. The system should handle failures gracefully. Notification content will be determined by the product team."
    },
    {
      "title": "In-App Messaging Center — v1",
      "type": "prd",
      "sections": ["overview", "requirements", "edge_cases"],
      "raw_text": "The in-app messaging center will surface messages from the platform and from other users. Messages must load quickly. The system will manage unread state. Ownership of the notification architecture is shared between backend and frontend. Edge cases will be handled appropriately. The UX team will finalize the interaction model."
    },
    {
      "title": "Email Digest Feature — Q3",
      "type": "implementation_brief",
      "sections": ["scope", "requirements"],
      "raw_text": "Users will receive a daily or weekly digest of activity from the platform. The content team will define what goes in the digest. Sending will be handled by the infrastructure team. The frequency should be customizable. Edge cases around unsubscribe behavior are to be determined. The digest should feel personal and relevant."
    }
  ]
}
```

**Expected behavior**:
- Individual spec analyses should each identify `vague_language`, `undefined_ownership`, `missing_acceptance_criteria`
- Portfolio review output should identify:
  - `repeated_ambiguity_categories`: ownership and acceptance criteria appearing across all three specs
  - `common_ownership_gaps`: notification architecture ownership is consistently unclear across the set
  - `top_risk_specs`: the spec with the highest combined severity
  - `portfolio_recommendations`: should recommend establishing a shared ownership model for notification infrastructure before any of the three specs proceed to engineering
- `confidence_band`: `high` (three specs is sufficient for cross-doc pattern analysis)
- `needs_human_review`: `false` if all three specs produce valid analyses

**Severity tendency**: The portfolio-level output does not use the same severity scale as individual specs. It identifies delivery risk at the portfolio level — expect output describing which patterns are systemic versus isolated.

---

## Test Summary Table

| Test | AI Called? | Expected Severity | Expected `needs_human_review` | Primary Categories |
|---|---|---|---|---|
| 1 — Vague Language PRD | Yes | `high` | `false` | vague_language, missing_acceptance_criteria, undefined_ownership, scope_boundary_gap |
| 2 — Missing Acceptance Criteria | Yes | `medium`–`high` | `false` | missing_acceptance_criteria, vague_language |
| 3 — Undefined Ownership | Yes | `high` | `false` | undefined_ownership, missing_decision_point |
| 4 — Contradictory Scope | Yes | `high`–`critical` | `true` | contradictory_requirement, scope_boundary_gap |
| 5 — Low Signal Short Spec | **No** | `low` | `true` | low_signal_noise |
| 6 — Malformed Model Output | Yes (fails) | N/A | `true` | N/A |
| 7 — Portfolio Review | Yes (multiple) | Portfolio-level | `false` | Cross-doc: ownership, acceptance_criteria |
