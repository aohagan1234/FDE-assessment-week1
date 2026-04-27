# Deliverable 2: Delegation Analysis

## Delegation Framework

Each FNOL processing task is classified into one of four categories:

| Category | Definition | Human Role | Agent Role |
|----------|------------|------------|------------|
| **Fully Agentic** | Agent executes autonomously; no human review required | None during execution; periodic audit | Full execution authority |
| **Agent-Led with Human Oversight** | Agent executes but flags for human review based on triggers | Review flagged cases; approve/override | Execute with escalation triggers |
| **Human-Led with Agent Support** | Human makes decision; agent provides data and recommendations | Decision authority | Data preparation; recommendation |
| **Human Only** | Agent has no role | Full execution | None |

---

## Codifiability Assessment Framework

For each task, we assess **codifiability**—the degree to which the decision logic can be expressed as explicit rules an AI agent can execute without ambiguity.

| Codifiability Level | Definition | Agent Suitability |
|---------------------|------------|-------------------|
| **HIGH** | Logic is fully expressible as deterministic rules with defined inputs and outputs. No interpretation required. | Fully agentic |
| **MEDIUM** | Logic can be expressed as rules + confidence thresholds. Some inputs may be ambiguous; clear escalation criteria exist. | Agent-led with oversight |
| **LOW** | Logic requires interpretation of context, policy nuance, or information not available to the agent. | Human-led or human-only |

---

## FNOL Processing Task Breakdown

### Task 1: Intake and Channel Normalization

**Classification**: Fully Agentic

**What it involves**:
- Receive FNOL from email inbox, phone transcript system, or web form endpoint
- Detect source channel
- Extract raw text content
- Create claim record with unique identifier and timestamp
- Trigger downstream processing

**Codifiability Assessment**: HIGH
- Channel detection: Deterministic based on source (email = EMAIL, transcription webhook = PHONE_TRANSCRIPT, CRM webhook = WEB_FORM)
- Record creation: Mechanical—generate UUID, capture timestamp, store raw content
- No interpretation required

**Delegation Justification**:

| Factor | Assessment | Rationale |
|--------|------------|-----------|
| **Risk level** | LOW | No judgment required; read-only data capture; errors detectable in next stage |
| **Ambiguity** | NONE | Channel detection is deterministic; content extraction is mechanical |
| **Compliance requirements** | NONE | No regulatory decision being made |
| **Cost of failure** | LOW | Missed intake would be detected by missing acknowledgment; retry possible |
| **Accountability needs** | LOW | Audit log of receipt is sufficient; no human sign-off required |

**Boundary justification**: This is pure data capture with HIGH codifiability. Every decision (channel type, timestamp, ID generation) follows deterministic rules. Human involvement would add latency without adding value.

---

### Task 2: Field Extraction from Unstructured Text

**Classification**: Agent-Led with Human Oversight

**What it involves**:
- Parse unstructured text (email body, phone transcript, web form narrative)
- Extract structured fields: claimant name, policy number, date of loss, loss description, loss location, injury indicators, damage estimate, third parties involved
- Assign confidence score to each extraction
- Flag extractions below confidence threshold for human review

**Codifiability Assessment**: MEDIUM
- Field extraction logic: Expressible as NLP patterns + entity recognition rules
- Confidence scoring: Quantifiable (0.00-1.00)
- Threshold for escalation: Codifiable (confidence < 0.90 for critical fields → escalate)
- **Limitation**: Input quality varies (phone transcripts may have gaps, garbled text); some fields may be implied rather than stated

**Delegation Justification**:

| Factor | Assessment | Rationale |
|--------|------------|-----------|
| **Risk level** | MEDIUM | Extraction errors propagate to coverage validation and routing; incorrect policy number = wrong coverage check |
| **Ambiguity** | MEDIUM | Unstructured text varies in quality; phone transcripts may have gaps; critical fields may be implied rather than stated |
| **Compliance requirements** | LOW | No regulatory requirement for extraction method; accuracy matters, not process |
| **Cost of failure** | MEDIUM | Incorrect extraction leads to downstream errors; but detectable before claimant impact |
| **Accountability needs** | MEDIUM | Need to know when extraction was wrong and why for improvement |

**Confidence thresholds**:
- **Critical fields** (policy_number, claimant_name, date_of_loss): Confidence ≥90% required to proceed; below 90% escalates to human
- **Non-critical fields** (loss_location, estimated_damage, third_parties): Confidence ≥70% to proceed; below 70% marked as LOW_CONFIDENCE but does not block

**Boundary justification**: Extraction has MEDIUM codifiability—rules can handle most cases, but input variance creates uncertainty. The confidence threshold mechanism makes the boundary explicit: proceed when confidence is high, escalate when low. This is not arbitrary—the 90% threshold is calibrated to acceptable downstream error rates.

---

### Task 3: Policy Lookup and Coverage Validation

**Classification**: Agent-Led with Human Oversight

**What it involves**:
- Query legacy policy administration system via SOAP endpoint using extracted policy number
- Retrieve policy details: status, coverage limits, exclusions, effective dates
- Determine coverage status: COVERED, NOT_COVERED, PARTIAL, LAPSED, REVIEW_REQUIRED
- Flag ambiguous coverage situations for human review

**Codifiability Assessment**: MEDIUM
- Policy lookup: Fully codifiable (SOAP request/response)
- Coverage determination for clear cases: Codifiable (policy ACTIVE, no exclusions match, date in range → COVERED)
- Coverage determination for edge cases: NOT codifiable (partial coverage, recent lapse, exclusion applicability require judgment)

**Decision Logic Codifiability**:
```
IF policy.status = ACTIVE 
   AND date_of_loss BETWEEN policy.effective_date AND policy.expiration_date
   AND loss_description does NOT match any policy.exclusion
THEN coverage_status = COVERED (codifiable)

IF policy.status = LAPSED within 30 days
THEN coverage_status = LAPSED, ESCALATE (edge case—human determines reinstatement)

IF loss_description PARTIALLY matches an exclusion
THEN coverage_status = PARTIAL, ESCALATE (judgment required)

IF coverage_status = NOT_COVERED
THEN ALWAYS ESCALATE (never auto-deny)
```

**Delegation Justification**:

| Factor | Assessment | Rationale |
|--------|------------|-----------|
| **Risk level** | HIGH | Coverage determination affects claim outcome; errors can lead to wrongful denial or fraud exposure |
| **Ambiguity** | MEDIUM-HIGH | Clear cases are codifiable; edge cases (partial coverage, exclusion applicability) require judgment |
| **Compliance requirements** | HIGH | Coverage determination may have regulatory implications; audit trail required |
| **Cost of failure** | HIGH | Incorrect NOT_COVERED = regulatory violation and customer harm; incorrect COVERED = fraud exposure |
| **Accountability needs** | HIGH | Must trace every coverage decision to source data and logic |

**Escalation triggers**:
- Policy not found → ESCALATE
- Policy status LAPSED within 30 days → ESCALATE
- Coverage status PARTIAL → ESCALATE
- Coverage status NOT_COVERED → ALWAYS ESCALATE (rule: agent never denies)
- Multiple policies for claimant → ESCALATE

**Boundary justification**: Coverage validation has MEDIUM codifiability for happy path, LOW codifiability for edge cases. The delegation boundary is drawn at determinism: clear COVERED proceeds; anything else escalates. The absolute rule—**agent never autonomously determines NOT_COVERED**—is the critical safety boundary. This is not arbitrary; it reflects regulatory and reputational risk.

---

### Task 4: Severity Triage

**Classification**: Agent-Led with Human Oversight

**What it involves**:
- Assess claim severity based on extracted data
- Assign severity tier: CRITICAL, STANDARD, LOW
- Route CRITICAL claims to immediate human attention regardless of other factors

**Codifiability Assessment**: MEDIUM-HIGH
- Keyword detection (fatality, injury, total loss): Fully codifiable
- Damage threshold rules: Fully codifiable
- Injury detection with uncertain language ("might be hurt"): Codifiable via confidence threshold

**Decision Logic Codifiability**:
```
IF fatality_indicated = true THEN severity = CRITICAL (codifiable)
IF injury_indicated = true THEN severity = CRITICAL (codifiable)
IF loss_description contains ["total loss", "destroyed", "collapsed"] THEN severity = CRITICAL (codifiable)
IF estimated_damage > $100,000 THEN severity = CRITICAL (codifiable)
IF estimated_damage > $25,000 OR third_parties = true THEN severity = STANDARD (codifiable)
ELSE severity = LOW (codifiable)

IF injury_indicated = true AND injury_confidence < 0.80 THEN ESCALATE (threshold-based)
IF severity = CRITICAL THEN ALWAYS ESCALATE (policy: human reviews all critical)
```

**Delegation Justification**:

| Factor | Assessment | Rationale |
|--------|------------|-----------|
| **Risk level** | MEDIUM-HIGH | Under-triaging delays response; over-triaging creates noise |
| **Ambiguity** | MEDIUM | Clear indicators are codifiable; borderline language ("might be hurt") requires confidence threshold |
| **Compliance requirements** | MEDIUM | Some jurisdictions require expedited handling of injury claims |
| **Cost of failure** | HIGH for false negatives | Missing a severe claim has outsized impact |
| **Accountability needs** | HIGH | Need audit trail for triage decisions |

**Boundary justification**: Severity triage has MEDIUM-HIGH codifiability—most indicators are deterministic. The delegation boundary is drawn at two points: (1) confidence threshold for ambiguous injury language, and (2) policy rule that all CRITICAL claims escalate. This reflects asymmetric error costs: missing a CRITICAL claim is worse than over-escalating.

---

### Task 5: Claim Categorization for Routing

**Classification**: Agent-Led with Human Oversight

**What it involves**:
- Determine claim category: AUTO, PROPERTY, LIABILITY, WORKERS_COMP, OTHER
- Category determines which adjuster pool receives the claim

**Codifiability Assessment**: MEDIUM
- Policy type → category mapping: Codifiable (AUTO policy = AUTO category)
- Loss description analysis: Partially codifiable (keyword matching, NLP classification)
- Multi-category claims: NOT codifiable (requires judgment)

**Decision Logic Codifiability**:
```
IF policy.type = AUTO AND loss_description indicates vehicle THEN category = AUTO (codifiable)
IF policy.type = HOMEOWNERS AND loss_description indicates property THEN category = PROPERTY (codifiable)
IF loss_description references multiple categories THEN ESCALATE (not codifiable)
IF category_confidence < 0.85 THEN ESCALATE (threshold-based)
```

**Delegation Justification**:

| Factor | Assessment | Rationale |
|--------|------------|-----------|
| **Risk level** | MEDIUM | Miscategorization causes routing error (current 18% rate suggests this is problematic) |
| **Ambiguity** | MEDIUM | Most claims map clearly; multi-category claims require judgment |
| **Compliance requirements** | LOW | No regulatory requirement for categorization method |
| **Cost of failure** | MEDIUM | Misroute requires rework (~15 min) but is recoverable |
| **Accountability needs** | MEDIUM | Track categorization accuracy for improvement |

**Boundary justification**: Categorization has MEDIUM codifiability. The 18% current routing error rate suggests this is non-trivial. The 85% confidence threshold and multi-category escalation rule make the boundary explicit. This is calibrated to reduce routing errors while maintaining throughput.

---

### Task 6: Adjuster Assignment

**Classification**: Fully Agentic

**What it involves**:
- Given a claim category and geography, select an adjuster from the appropriate pool
- Apply assignment rules (round-robin, workload balancing, specialization matching)
- Create assignment in CRM

**Codifiability Assessment**: HIGH
- Category → adjuster pool mapping: Fully codifiable
- Geography matching: Fully codifiable (claim location → adjuster territory)
- Selection algorithm: Fully codifiable (e.g., lowest current queue depth)

**Decision Logic Codifiability**:
```
SELECT adjuster WHERE 
  category IN adjuster.specializations
  AND claim.location.state IN adjuster.territories
  AND adjuster.active = true
ORDER BY current_queue_depth ASC
LIMIT 1

IF no eligible adjuster THEN ESCALATE
```

**Delegation Justification**:

| Factor | Assessment | Rationale |
|--------|------------|-----------|
| **Risk level** | LOW | Once category is correct, adjuster selection is mechanical |
| **Ambiguity** | LOW | Assignment rules are deterministic |
| **Compliance requirements** | NONE | No regulatory requirement |
| **Cost of failure** | LOW | Wrong adjuster within correct pool is minor inconvenience |
| **Accountability needs** | LOW | Audit log sufficient |

**Boundary justification**: Adjuster assignment has HIGH codifiability. Once categorization is validated, selection follows deterministic rules. Human involvement would add no value.

---

### Task 7: Claimant Acknowledgment

**Classification**: Fully Agentic

**What it involves**:
- Generate acknowledgment communication (email)
- Include: claim number, receipt confirmation, next steps, expected response time, contact information
- Send via CRM email integration
- Log acknowledgment timestamp

**Codifiability Assessment**: HIGH
- Template selection: Fully codifiable (one standard template)
- Variable substitution: Fully codifiable (claim_id, claimant_name, timestamp)
- Send logic: Fully codifiable (if claimant_email exists, send)

**Decision Logic Codifiability**:
```
IF claimant_email IS NOT NULL AND acknowledged_at IS NULL THEN
  content = RENDER_TEMPLATE("fnol_acknowledgment", variables)
  VALIDATE content does not contain forbidden terms (coverage status, denial, approval)
  IF validation passes THEN SEND and SET acknowledged_at = NOW()
  ELSE ESCALATE (template validation failed)
```

**Content guardrails** (codifiable):
- Must NOT include coverage status
- Must NOT include claim amount estimates
- Must NOT include adjuster personal contact
- Must include claim number and general contact

**Delegation Justification**:

| Factor | Assessment | Rationale |
|--------|------------|-----------|
| **Risk level** | LOW | Template-based; no sensitive determinations |
| **Ambiguity** | NONE | Content is standardized |
| **Compliance requirements** | LOW | Must acknowledge within SLA; content is standardized |
| **Cost of failure** | MEDIUM | Missing acknowledgment violates SLA |
| **Accountability needs** | LOW | Timestamp log sufficient |

**Boundary justification**: Acknowledgment has HIGH codifiability. It is template-based variable substitution with codifiable guardrails. No judgment required.

---

### Task 8: High-Value Claim Review

**Classification**: Human-Led with Agent Support

**What it involves**:
- Review claims where estimated_damage > $50,000 or policy_limit > $500,000
- Validate coverage determination
- Confirm routing decision
- Approve or modify severity classification

**Codifiability Assessment**: LOW
- Threshold trigger: Codifiable (damage > $50,000 → flag)
- Review decision: NOT codifiable (requires human judgment on coverage nuances, fraud signals, policy interpretation)

**Delegation Justification**:

| Factor | Assessment | Rationale |
|--------|------------|-----------|
| **Risk level** | HIGH | Financial exposure justifies human judgment |
| **Ambiguity** | VARIABLE | Some high-value claims are straightforward; others require judgment |
| **Compliance requirements** | HIGH | Large claims may have additional regulatory requirements |
| **Cost of failure** | VERY HIGH | Errors on high-value claims have outsized impact |
| **Accountability needs** | VERY HIGH | Named human must approve |

**Agent role**: Pre-populate all data; provide coverage result with rationale; provide routing recommendation; highlight anomalies. Human makes final determination.

**Boundary justification**: High-value review has LOW codifiability for the decision itself. The trigger is codifiable; the judgment is not. This is the explicit client requirement, and it reflects appropriate risk allocation for significant financial exposure.

---

### Task 9: Ambiguous Claim Resolution

**Classification**: Human-Led with Agent Support

**What it involves**:
- Review claims escalated due to low confidence on any processing step
- Review claims with conflicting or incomplete information
- Make determination that agent could not make

**Codifiability Assessment**: LOW
- Escalation trigger: Codifiable (confidence < threshold, coverage ambiguous, etc.)
- Resolution: NOT codifiable (by definition, these are cases where rules don't apply)

**Delegation Justification**:

| Factor | Assessment | Rationale |
|--------|------------|-----------|
| **Risk level** | VARIABLE | Depends on nature of ambiguity |
| **Ambiguity** | HIGH | By definition, these are non-codifiable cases |
| **Compliance requirements** | VARIABLE | Depends on ambiguity type |
| **Cost of failure** | VARIABLE | Human judgment warranted when agent cannot proceed |
| **Accountability needs** | HIGH | Human must own resolution |

**Agent role**: Present all data; explain escalation trigger; provide options if applicable; log human decision.

**Boundary justification**: Ambiguous resolution has LOW codifiability by definition. The agent's role is detection and data preparation; the human's role is judgment. The confidence thresholds that trigger escalation are the mechanism that makes this boundary explicit and testable.

---

## Delegation Summary

| Task | Classification | Codifiability | Key Escalation Trigger |
|------|----------------|---------------|------------------------|
| 1. Intake | Fully Agentic | HIGH | System failure only |
| 2. Extraction | Agent-Led + Oversight | MEDIUM | Confidence <90% on critical fields |
| 3. Coverage Validation | Agent-Led + Oversight | MEDIUM | Any non-COVERED result or low confidence |
| 4. Severity Triage | Agent-Led + Oversight | MEDIUM-HIGH | CRITICAL severity or injury confidence <80% |
| 5. Categorization | Agent-Led + Oversight | MEDIUM | Confidence <85% or multi-category |
| 6. Adjuster Assignment | Fully Agentic | HIGH | No eligible adjuster |
| 7. Acknowledgment | Fully Agentic | HIGH | Template validation failure |
| 8. High-Value Review | Human-Led + Support | LOW (decision) | Threshold trigger (codifiable) |
| 9. Ambiguous Resolution | Human-Led + Support | LOW | Confidence thresholds (codifiable) |

---

## Delegation Boundary Principles

1. **Codifiability determines automation ceiling**: HIGH codifiability → fully agentic; MEDIUM → agent-led with oversight; LOW → human-led.

2. **Asymmetric error costs adjust thresholds**: When false negatives are more costly than false positives, thresholds are conservative (e.g., CRITICAL severity always escalates).

3. **Never automate coverage denial**: The agent can determine COVERED with high confidence. It can never autonomously determine NOT_COVERED because the regulatory and reputational cost of error is unacceptable.

4. **Confidence thresholds make boundaries testable**: Every escalation trigger is quantifiable (90%, 85%, 80%). This allows validation that boundaries work as intended.
