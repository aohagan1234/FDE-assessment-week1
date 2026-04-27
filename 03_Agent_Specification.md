# Deliverable 3: Agent Specification

## 1. Purpose

The FNOL Processing Agent automates the intake, validation, triage, routing, and acknowledgment of First Notice of Loss claims. The agent processes unstructured claim submissions from multiple channels, extracts structured data, validates coverage against policy records, triages by severity, routes to appropriate adjusters, and acknowledges receipt to claimants—all within the 2-hour SLA. Claims requiring human judgment are escalated with pre-populated data and clear escalation rationale.

---

## 2. Scope

### In-Scope

1. Receiving FNOL submissions from three channels: email, phone transcripts, web forms
2. Extracting structured claim data from unstructured text
3. Validating coverage against the policy administration system
4. Triaging claims by severity (CRITICAL, STANDARD, LOW)
5. Categorizing claims for routing (AUTO, PROPERTY, LIABILITY, WORKERS_COMP, OTHER)
6. Assigning claims to adjuster queues
7. Sending acknowledgment communications to claimants
8. Escalating claims that meet defined triggers to human review queue
9. Logging all actions for audit trail
10. Providing real-time dashboard metrics

### Out-of-Scope

1. Making coverage denial determinations (agent escalates; human decides)
2. Adjudicating claim validity or fraud detection
3. Determining claim payout amounts
4. Communicating claim decisions (approval/denial) to claimants
5. Handling claim amendments or supplements after initial FNOL
6. Managing adjuster workloads or capacity planning
7. Modifying policy administration system records
8. Training or retraining extraction models
9. Integrating with payment systems

### Explicit Exclusions (Guard Rails)

The agent must NOT:
1. Send any communication containing coverage status to claimants
2. Mark any claim as NOT_COVERED without human approval
3. Bypass escalation triggers for any reason
4. Modify or delete policy records
5. Process claims for which policy lookup fails without escalation
6. Assign CRITICAL severity claims without escalation
7. Override human decisions on escalated claims

---

## 3. Users and Stakeholders

| Role | Interaction with Agent | Needs |
|------|------------------------|-------|
| **Claims Specialists** (12) | Review escalated claims; approve/override agent decisions; monitor queue | Clear escalation reasons; complete data; easy override interface |
| **Claims Supervisor** | Monitor throughput and SLA compliance; manage escalation queue depth; configure thresholds | Real-time dashboard; alert configuration; trend reporting |
| **Adjusters** | Receive routed claims with complete packages | Accurate categorization; complete claim data; contact info |
| **Claimants** | Submit FNOL; receive acknowledgment | Timely acknowledgment; clear claim number; next steps |
| **IT Operations** | Monitor system health; manage integrations | Integration status; error rates; latency metrics |
| **Compliance/Audit** | Review decision audit trails | Complete logging; decision rationale; retention compliance |

---

## 4. Entity Definitions

### Entity: Claim

| Attribute | Type | Constraints | Notes |
|-----------|------|-------------|-------|
| `claim_id` | UUID | Primary key, immutable | Generated on intake |
| `external_reference` | string | Max 100 chars, nullable | Reference from source channel |
| `channel` | enum | [EMAIL, PHONE_TRANSCRIPT, WEB_FORM], required | Source detection is deterministic |
| `raw_content` | text | Required, max 50KB | Original unstructured input |
| `received_at` | timestamp | UTC, ISO 8601, required, immutable | SLA clock starts here |
| `status` | enum | See state machine below, required | Current processing state |
| `policy_number` | string | Max 50 chars, nullable | Extracted field |
| `policy_number_confidence` | decimal | 0.00-1.00, nullable | Extraction confidence |
| `claimant_name` | string | Max 200 chars, nullable | Extracted field |
| `claimant_name_confidence` | decimal | 0.00-1.00, nullable | Extraction confidence |
| `claimant_email` | string | RFC 5322 format, nullable | For acknowledgment |
| `claimant_phone` | string | Max 20 chars, nullable | Contact |
| `date_of_loss` | date | ISO 8601, nullable | Extracted field |
| `date_of_loss_confidence` | decimal | 0.00-1.00, nullable | Extraction confidence |
| `loss_description` | text | Max 10KB, nullable | Extracted field |
| `loss_location` | string | Max 500 chars, nullable | Extracted field |
| `injury_indicated` | boolean | Default false | Extracted indicator |
| `injury_confidence` | decimal | 0.00-1.00, nullable | Extraction confidence |
| `fatality_indicated` | boolean | Default false | Extracted indicator |
| `estimated_damage` | decimal | Nullable, >=0 | In USD |
| `third_parties_involved` | boolean | Default false | Extracted indicator |
| `extraction_confidence` | decimal | 0.00-1.00, nullable | Minimum of critical field confidences |
| `severity` | enum | [CRITICAL, STANDARD, LOW], nullable | Assigned after triage |
| `severity_confidence` | decimal | 0.00-1.00, nullable | Triage confidence |
| `category` | enum | [AUTO, PROPERTY, LIABILITY, WORKERS_COMP, OTHER], nullable | Assigned after categorization |
| `category_confidence` | decimal | 0.00-1.00, nullable | Categorization confidence |
| `coverage_status` | enum | [COVERED, NOT_COVERED, PARTIAL, LAPSED, REVIEW_REQUIRED], nullable | From validation |
| `coverage_details` | JSON | Nullable | Policy details snapshot |
| `assigned_adjuster_id` | UUID | Foreign key to Adjuster, nullable | Adjuster assignment |
| `acknowledged_at` | timestamp | UTC, ISO 8601, nullable | Acknowledgment sent timestamp |
| `escalation_reason` | string | Max 500 chars, nullable | Why escalated |
| `escalated_at` | timestamp | UTC, ISO 8601, nullable | When escalated |
| `escalation_priority` | enum | [LOW, MEDIUM, HIGH, URGENT], nullable | Escalation priority |
| `resolved_by` | UUID | Foreign key to User, nullable | Specialist who resolved |
| `resolved_at` | timestamp | UTC, ISO 8601, nullable | Resolution timestamp |
| `completed_at` | timestamp | UTC, ISO 8601, nullable | Final completion timestamp |
| `created_at` | timestamp | UTC, ISO 8601, immutable | Record creation |
| `updated_at` | timestamp | UTC, ISO 8601 | Last modification |
| `created_by` | string | System identifier, immutable | "fnol-agent-v1" or user ID |
| `updated_by` | string | System identifier | Last modifier |

**Foreign Key Behavior**:
- `assigned_adjuster_id` → Adjuster: ON DELETE SET NULL (claim remains, adjuster reference cleared)
- `resolved_by` → User: ON DELETE RESTRICT (cannot delete user with resolved claims)

**Claim State Machine**:

```
RECEIVED
  │
  └─► EXTRACTING
        │
        ├─► VALIDATING (extraction confidence >= 0.90 on critical fields)
        │     │
        │     ├─► TRIAGING (coverage_status = COVERED)
        │     │     │
        │     │     ├─► ROUTING (severity != CRITICAL and triage confidence met)
        │     │     │     │
        │     │     │     ├─► PENDING_ACKNOWLEDGMENT (adjuster assigned)
        │     │     │     │     │
        │     │     │     │     └─► COMPLETED (acknowledgment sent)
        │     │     │     │
        │     │     │     └─► ESCALATED (no eligible adjuster or category confidence < 0.85)
        │     │     │
        │     │     └─► ESCALATED (severity = CRITICAL or injury confidence < 0.80)
        │     │
        │     └─► ESCALATED (coverage not COVERED or policy not found)
        │
        └─► ESCALATED (extraction confidence < 0.90 on critical field)

ESCALATED
  │
  ├─► ROUTING (human approves, releases to automated flow)
  │
  └─► COMPLETED (human resolves directly)

COMPLETED is terminal.
FAILED is terminal.
```

**State Transition Table**:

| From State | To State | Trigger | Prerequisites |
|------------|----------|---------|---------------|
| RECEIVED | EXTRACTING | Agent begins processing | raw_content not empty |
| EXTRACTING | VALIDATING | Extraction complete | extraction_confidence >= 0.90 on all critical fields |
| EXTRACTING | ESCALATED | Low confidence | extraction_confidence < 0.90 on any critical field after retry |
| VALIDATING | TRIAGING | Coverage validated | coverage_status = COVERED |
| VALIDATING | ESCALATED | Coverage issue | coverage_status IN [NOT_COVERED, PARTIAL, LAPSED, REVIEW_REQUIRED] OR policy not found |
| TRIAGING | ROUTING | Triage complete | severity assigned AND severity != CRITICAL AND (injury_indicated = false OR injury_confidence >= 0.80) |
| TRIAGING | ESCALATED | Critical or uncertain injury | severity = CRITICAL OR (injury_indicated = true AND injury_confidence < 0.80) |
| ROUTING | PENDING_ACKNOWLEDGMENT | Adjuster assigned | assigned_adjuster_id IS NOT NULL AND category_confidence >= 0.85 |
| ROUTING | ESCALATED | Cannot route | No eligible adjuster OR category_confidence < 0.85 |
| PENDING_ACKNOWLEDGMENT | COMPLETED | Acknowledgment sent | acknowledged_at IS NOT NULL |
| ESCALATED | ROUTING | Human release | Human approves continuation; resolution logged |
| ESCALATED | COMPLETED | Human resolution | Human completes directly; resolution logged |
| Any | FAILED | System error | Unrecoverable error after all retries exhausted |

---

### Entity: Policy (Read from External System)

| Attribute | Type | Notes |
|-----------|------|-------|
| `policy_number` | string | Primary key in external system |
| `policy_status` | enum | [ACTIVE, LAPSED, CANCELLED, PENDING] |
| `policyholder_name` | string | For verification |
| `effective_date` | date | ISO 8601, coverage start |
| `expiration_date` | date | ISO 8601, coverage end |
| `policy_type` | enum | [AUTO, HOMEOWNERS, COMMERCIAL, LIABILITY, WORKERS_COMP] |
| `coverage_limits` | JSON | Limit amounts by coverage type |
| `deductible` | decimal | Policy deductible in USD |
| `exclusions` | array of string | List of exclusion codes |

*Note: Read-only. Agent queries via SOAP; does not write to policy administration system.*

---

### Entity: Adjuster

| Attribute | Type | Constraints | Notes |
|-----------|------|-------------|-------|
| `adjuster_id` | UUID | Primary key | |
| `name` | string | Max 200 chars, required | Display name |
| `email` | string | RFC 5322 format, required | Contact email |
| `specializations` | array of enum | [AUTO, PROPERTY, LIABILITY, WORKERS_COMP], min 1 | Claim categories handled |
| `territories` | array of string | State codes, min 1 | Geographic regions served |
| `active` | boolean | Required, default true | Available for assignment |
| `current_queue_depth` | integer | >= 0, default 0 | Claims in queue |
| `created_at` | timestamp | UTC, ISO 8601, immutable | |
| `updated_at` | timestamp | UTC, ISO 8601 | |

---

### Entity: Escalation

| Attribute | Type | Constraints | Notes |
|-----------|------|-------------|-------|
| `escalation_id` | UUID | Primary key, immutable | |
| `claim_id` | UUID | Foreign key to Claim, required | |
| `reason_code` | enum | See codes below, required | |
| `reason_detail` | string | Max 500 chars, required | Human-readable explanation |
| `priority` | enum | [LOW, MEDIUM, HIGH, URGENT], required | |
| `status` | enum | [OPEN, ASSIGNED, IN_REVIEW, RESOLVED], default OPEN | |
| `assigned_to` | UUID | Foreign key to User, nullable | Specialist assigned |
| `resolution_action` | enum | [APPROVED, MODIFIED, REJECTED], nullable | Human decision |
| `resolution_notes` | string | Max 1000 chars, nullable | Human notes |
| `created_at` | timestamp | UTC, ISO 8601, immutable | |
| `resolved_at` | timestamp | UTC, ISO 8601, nullable | |
| `created_by` | string | "fnol-agent-v1", immutable | |
| `resolved_by` | UUID | Foreign key to User, nullable | |

**Foreign Key Behavior**:
- `claim_id` → Claim: ON DELETE CASCADE (escalation deleted if claim deleted)
- `assigned_to` → User: ON DELETE SET NULL
- `resolved_by` → User: ON DELETE RESTRICT

**Escalation Reason Codes**:
| Code | Priority | Description |
|------|----------|-------------|
| `LOW_EXTRACTION_CONFIDENCE` | MEDIUM | Critical field confidence < 0.90 |
| `POLICY_NOT_FOUND` | HIGH | Policy number not in system |
| `POLICY_LAPSED` | HIGH | Policy lapsed within 30 days |
| `COVERAGE_NOT_COVERED` | HIGH | Coverage determination is NOT_COVERED |
| `COVERAGE_PARTIAL` | MEDIUM | Partial coverage; exclusion may apply |
| `COVERAGE_REVIEW_REQUIRED` | MEDIUM | Coverage ambiguous |
| `CRITICAL_SEVERITY` | URGENT | Severity = CRITICAL |
| `INJURY_LOW_CONFIDENCE` | HIGH | Injury indicated but confidence < 0.80 |
| `CATEGORY_LOW_CONFIDENCE` | LOW | Category confidence < 0.85 |
| `MULTI_CATEGORY_CLAIM` | LOW | Claim spans multiple categories |
| `HIGH_VALUE_CLAIM` | HIGH | Damage > $50K or policy limit > $500K |
| `NO_ELIGIBLE_ADJUSTER` | MEDIUM | No adjuster matches category/territory |
| `SYSTEM_ERROR` | HIGH | Integration failure after retries |
| `SLA_AT_RISK` | URGENT | Claim unacknowledged at 90 minutes |

---

## 5. Inputs

### 5.1 Email FNOL

| Field | Source | Format | Notes |
|-------|--------|--------|-------|
| From address | Email header | RFC 5322 | Claimant email |
| Subject | Email header | String | May contain policy number |
| Body | Email content | Plain text or HTML | Primary extraction source |
| Attachments | Email attachments | Various | Stored in DMS; not parsed in v1 |
| Received timestamp | Email header | RFC 2822 | SLA clock start |

**Ingestion method**: Poll configured mailbox every 60 seconds OR receive via webhook from email service

### 5.2 Phone Transcript FNOL

| Field | Source | Format | Notes |
|-------|--------|--------|-------|
| Transcript text | Transcription service | String | Full call transcript |
| Call timestamp | Transcription metadata | ISO 8601 | When call occurred |
| Caller phone | Transcription metadata | String | Caller ID |
| Call duration | Transcription metadata | Integer (seconds) | For quality assessment |
| Agent ID | Transcription metadata | String | Call center agent |

**Ingestion method**: Webhook from transcription service (push)

### 5.3 Web Form FNOL

| Field | Source | Format | Notes |
|-------|--------|--------|-------|
| Form fields | CRM webhook | JSON | Structured; may have free-text |
| Submission timestamp | CRM | ISO 8601 | SLA clock start |
| Session ID | CRM | String | For deduplication |

**Ingestion method**: Webhook from CRM (push)

### 5.4 Policy Data (SOAP API)

| Field | Source | Format |
|-------|--------|--------|
| Policy record | Policy Admin System | XML (SOAP response) |
| Lookup key | Agent-provided | policy_number string |

**Integration contract**: See Section 8

---

## 6. Outputs

### 6.1 Claim Record

**Destination**: CRM via API (see Section 8.2)

**Content**: Full Claim entity as defined in Section 4

### 6.2 Acknowledgment Email

**Destination**: Claimant via CRM email service

**Template ID**: `fnol_acknowledgment`

**Template Variables**:
| Variable | Source | Default if Missing |
|----------|--------|-------------------|
| `{{claimant_name}}` | claim.claimant_name | "Valued Customer" |
| `{{claim_id}}` | claim.claim_id | (required—never missing) |
| `{{received_date}}` | FORMAT(claim.received_at, "MMMM D, YYYY") | (required) |
| `{{next_steps}}` | CONFIG.STANDARD_NEXT_STEPS | (required) |
| `{{contact_phone}}` | CONFIG.CLAIMS_PHONE | (required) |
| `{{contact_hours}}` | CONFIG.BUSINESS_HOURS | (required) |

**Content Guardrails** (validated before send):
- Must NOT contain: "covered", "not covered", "denied", "approved", "coverage"
- Must NOT contain: estimated_damage value
- Must NOT contain: adjuster personal email or direct phone
- Must contain: claim_id
- Must contain: contact_phone

### 6.3 Escalation Record

**Destination**: Escalation queue in CRM

**Content**: Full Escalation entity as defined in Section 4, plus snapshot of all extracted claim data

### 6.4 Adjuster Assignment

**Destination**: CRM adjuster queue

| Field | Source |
|-------|--------|
| claim_id | claim.claim_id |
| adjuster_id | Selected adjuster |
| assignment_timestamp | NOW() |
| claim_package | All extracted data + coverage details + documents |

### 6.5 Audit Log Entry

**Destination**: Logging system (append-only)

| Field | Type | Description |
|-------|------|-------------|
| log_id | UUID | Unique identifier |
| timestamp | ISO 8601 UTC | Event time |
| event_type | enum | See event types below |
| claim_id | UUID | Claim reference |
| agent_id | string | "fnol-agent-v1" |
| input_snapshot | JSON | Relevant input (max 10KB) |
| output_snapshot | JSON | Resulting output (max 10KB) |
| confidence_scores | JSON | Any confidence values |
| latency_ms | integer | Processing time |
| error | string | Error details if any |

**Event Types**:
`CLAIM_RECEIVED`, `EXTRACTION_STARTED`, `EXTRACTION_COMPLETED`, `EXTRACTION_RETRY`, `POLICY_LOOKUP_STARTED`, `POLICY_LOOKUP_COMPLETED`, `POLICY_LOOKUP_FAILED`, `COVERAGE_VALIDATED`, `SEVERITY_ASSIGNED`, `CATEGORY_ASSIGNED`, `ADJUSTER_ASSIGNED`, `ACKNOWLEDGMENT_SENT`, `ACKNOWLEDGMENT_FAILED`, `ESCALATION_CREATED`, `ESCALATION_RESOLVED`, `HUMAN_OVERRIDE`, `CLAIM_COMPLETED`, `STATE_TRANSITION`

---

## 7. Decision Logic

### 7.1 Channel Detection and Intake

```
REQUIREMENT 7.1.1: Intake Processing
WHEN message received on configured endpoint
THEN:
  IF source = email inbox (IMAP poll or webhook)
    THEN channel = EMAIL
  ELSE IF source = transcription service webhook
    THEN channel = PHONE_TRANSCRIPT
  ELSE IF source = CRM form webhook
    THEN channel = WEB_FORM
  ELSE
    REJECT with error "Unknown channel"
  
  CREATE Claim with:
    claim_id = UUID.generate()
    channel = detected channel
    raw_content = message content
    received_at = NOW()
    status = RECEIVED
    created_at = NOW()
    created_by = "fnol-agent-v1"
  
  LOG event CLAIM_RECEIVED
  TRANSITION status to EXTRACTING
```

### 7.2 Field Extraction

```
REQUIREMENT 7.2.1: Extract Fields
WHEN claim.status = EXTRACTING
THEN:
  CALL extraction_service(claim.raw_content, claim.channel)
  
  FOR each field IN [policy_number, claimant_name, claimant_email, claimant_phone,
                     date_of_loss, loss_description, loss_location, injury_indicated,
                     fatality_indicated, estimated_damage, third_parties_involved]:
    SET claim.{field} = extracted_value
    SET claim.{field}_confidence = extraction_confidence (where applicable)
  
  SET claim.extraction_confidence = MIN(
    claim.policy_number_confidence,
    claim.claimant_name_confidence,
    claim.date_of_loss_confidence
  )
  
  LOG event EXTRACTION_COMPLETED with confidence scores
```

```
REQUIREMENT 7.2.2: Extraction Confidence Check
DEFINE critical_fields = [policy_number, claimant_name, date_of_loss]
DEFINE critical_threshold = 0.90

IF claim.extraction_confidence >= critical_threshold
  THEN TRANSITION status to VALIDATING

ELSE:
  LOG event EXTRACTION_RETRY
  RETRY extraction with alternative prompt (max 1 retry)
  
  IF after retry, extraction_confidence still < critical_threshold
    THEN:
      SET claim.status = ESCALATED
      SET claim.escalation_reason = "LOW_EXTRACTION_CONFIDENCE"
      SET claim.escalation_priority = MEDIUM
      SET claim.escalated_at = NOW()
      CREATE Escalation with reason_code = LOW_EXTRACTION_CONFIDENCE
      LOG event ESCALATION_CREATED
```

### 7.3 Policy Lookup and Coverage Validation

```
REQUIREMENT 7.3.1: Policy Lookup
WHEN claim.status = VALIDATING
THEN:
  LOG event POLICY_LOOKUP_STARTED
  CALL PolicyAdminSystem.GetPolicy(claim.policy_number)
  
  IF response = NOT_FOUND
    THEN:
      SET claim.coverage_status = REVIEW_REQUIRED
      TRANSITION to ESCALATED with reason_code = POLICY_NOT_FOUND, priority = HIGH
      LOG event POLICY_LOOKUP_FAILED
  
  ELSE IF response = TIMEOUT or ERROR
    THEN:
      RETRY up to 2 times with backoff (2s, 4s)
      IF all retries fail:
        TRANSITION to ESCALATED with reason_code = SYSTEM_ERROR, priority = HIGH
        LOG event POLICY_LOOKUP_FAILED
  
  ELSE:
    LOG event POLICY_LOOKUP_COMPLETED
    PROCEED to coverage validation
```

```
REQUIREMENT 7.3.2: Coverage Determination
WHEN policy found
THEN:
  IF policy.status = LAPSED AND (NOW() - policy.expiration_date) <= 30 days
    THEN:
      SET claim.coverage_status = LAPSED
      TRANSITION to ESCALATED with reason_code = POLICY_LAPSED, priority = HIGH
  
  ELSE IF policy.status IN [CANCELLED, PENDING]
    THEN:
      SET claim.coverage_status = REVIEW_REQUIRED
      TRANSITION to ESCALATED with reason_code = COVERAGE_REVIEW_REQUIRED, priority = MEDIUM
  
  ELSE IF policy.status = ACTIVE
    THEN:
      IF claim.date_of_loss NOT BETWEEN policy.effective_date AND policy.expiration_date
        THEN:
          SET claim.coverage_status = NOT_COVERED
          TRANSITION to ESCALATED with reason_code = COVERAGE_NOT_COVERED, priority = HIGH
      
      ELSE IF loss_matches_exclusion(claim.loss_description, policy.exclusions) = PARTIAL_MATCH
        THEN:
          SET claim.coverage_status = PARTIAL
          TRANSITION to ESCALATED with reason_code = COVERAGE_PARTIAL, priority = MEDIUM
      
      ELSE IF loss_matches_exclusion(claim.loss_description, policy.exclusions) = FULL_MATCH
        THEN:
          SET claim.coverage_status = NOT_COVERED
          TRANSITION to ESCALATED with reason_code = COVERAGE_NOT_COVERED, priority = HIGH
      
      ELSE:
          SET claim.coverage_status = COVERED
          SET claim.coverage_details = {
            policy_type: policy.policy_type,
            coverage_limits: policy.coverage_limits,
            deductible: policy.deductible
          }
          TRANSITION status to TRIAGING
          LOG event COVERAGE_VALIDATED
```

### 7.4 Severity Triage

```
REQUIREMENT 7.4.1: Severity Assignment
WHEN claim.status = TRIAGING
THEN:
  IF claim.fatality_indicated = true
    THEN SET claim.severity = CRITICAL, claim.severity_confidence = 1.0
  
  ELSE IF claim.injury_indicated = true
    THEN SET claim.severity = CRITICAL, claim.severity_confidence = claim.injury_confidence
  
  ELSE IF contains_critical_keywords(claim.loss_description)
    -- Keywords: "total loss", "totaled", "destroyed", "collapsed", 
    --          "major structural", "fatality", "death", "killed", "deceased"
    THEN SET claim.severity = CRITICAL, claim.severity_confidence = 0.85
  
  ELSE IF claim.estimated_damage > 100000
    THEN SET claim.severity = CRITICAL, claim.severity_confidence = 0.95
  
  ELSE IF claim.estimated_damage > 25000 OR claim.third_parties_involved = true
    THEN SET claim.severity = STANDARD, claim.severity_confidence = 0.90
  
  ELSE
    SET claim.severity = LOW, claim.severity_confidence = 0.90
  
  LOG event SEVERITY_ASSIGNED
```

```
REQUIREMENT 7.4.2: Severity Escalation
IF claim.severity = CRITICAL
  THEN TRANSITION to ESCALATED with reason_code = CRITICAL_SEVERITY, priority = URGENT

ELSE IF claim.injury_indicated = true AND claim.injury_confidence < 0.80
  THEN TRANSITION to ESCALATED with reason_code = INJURY_LOW_CONFIDENCE, priority = HIGH

ELSE
  PROCEED to categorization
```

### 7.5 Claim Categorization

```
REQUIREMENT 7.5.1: Category Assignment
WHEN severity assigned AND not escalated
THEN:
  primary_category = infer_category(claim.loss_description, policy.policy_type)
  
  IF policy.policy_type = AUTO AND loss_indicates_vehicle(claim.loss_description)
    THEN SET claim.category = AUTO, claim.category_confidence = 0.95
  
  ELSE IF policy.policy_type = HOMEOWNERS AND loss_indicates_property(claim.loss_description)
    THEN SET claim.category = PROPERTY, claim.category_confidence = 0.95
  
  ELSE IF policy.policy_type = LIABILITY OR loss_indicates_liability(claim.loss_description)
    THEN SET claim.category = LIABILITY, claim.category_confidence = 0.90
  
  ELSE IF policy.policy_type = WORKERS_COMP OR loss_indicates_workplace(claim.loss_description)
    THEN SET claim.category = WORKERS_COMP, claim.category_confidence = 0.90
  
  ELSE
    SET claim.category = OTHER, claim.category_confidence = 0.70
  
  LOG event CATEGORY_ASSIGNED
```

```
REQUIREMENT 7.5.2: Category Escalation
IF claim.category_confidence < 0.85
  THEN TRANSITION to ESCALATED with reason_code = CATEGORY_LOW_CONFIDENCE, priority = LOW

ELSE IF loss_spans_multiple_categories(claim.loss_description)
  THEN TRANSITION to ESCALATED with reason_code = MULTI_CATEGORY_CLAIM, priority = LOW

ELSE
  TRANSITION status to ROUTING
```

### 7.6 High-Value Check

```
REQUIREMENT 7.6.1: High-Value Threshold
WHEN claim.status = ROUTING
THEN:
  IF claim.estimated_damage > 50000
    THEN TRANSITION to ESCALATED with reason_code = HIGH_VALUE_CLAIM, priority = HIGH
  
  ELSE IF policy.coverage_limits.total > 500000
    THEN TRANSITION to ESCALATED with reason_code = HIGH_VALUE_CLAIM, priority = HIGH
  
  ELSE
    PROCEED to adjuster assignment
```

### 7.7 Adjuster Assignment

```
REQUIREMENT 7.7.1: Select Adjuster
WHEN high-value check passed
THEN:
  eligible_adjusters = SELECT FROM adjusters WHERE:
    claim.category IN adjuster.specializations
    AND claim.loss_location.state IN adjuster.territories
    AND adjuster.active = true
  
  IF eligible_adjusters.count = 0
    THEN TRANSITION to ESCALATED with reason_code = NO_ELIGIBLE_ADJUSTER, priority = MEDIUM
  
  ELSE:
    selected = eligible_adjusters.ORDER_BY(current_queue_depth ASC).FIRST()
    SET claim.assigned_adjuster_id = selected.adjuster_id
    INCREMENT selected.current_queue_depth
    CALL CRM.AssignClaim(claim.claim_id, selected.adjuster_id)
    TRANSITION status to PENDING_ACKNOWLEDGMENT
    LOG event ADJUSTER_ASSIGNED
```

### 7.8 Acknowledgment

```
REQUIREMENT 7.8.1: Send Acknowledgment
WHEN claim.status = PENDING_ACKNOWLEDGMENT OR claim.status = ESCALATED
AND claim.acknowledged_at IS NULL
AND claim.claimant_email IS NOT NULL
THEN:
  content = RENDER_TEMPLATE("fnol_acknowledgment", {
    claimant_name: claim.claimant_name OR "Valued Customer",
    claim_id: claim.claim_id,
    received_date: FORMAT(claim.received_at, "MMMM D, YYYY"),
    next_steps: CONFIG.STANDARD_NEXT_STEPS,
    contact_phone: CONFIG.CLAIMS_PHONE,
    contact_hours: CONFIG.BUSINESS_HOURS
  })
  
  -- Validate content before sending
  IF content CONTAINS ANY OF ["covered", "not covered", "denied", "approved", "coverage"]
    THEN:
      LOG event ACKNOWLEDGMENT_FAILED with reason "forbidden terms"
      ALERT operations
      DO NOT SEND
      RETURN
  
  IF content DOES NOT CONTAIN claim.claim_id
    THEN:
      LOG event ACKNOWLEDGMENT_FAILED with reason "missing claim_id"
      ALERT operations
      DO NOT SEND
      RETURN
  
  CALL EmailService.Send(claim.claimant_email, content)
  SET claim.acknowledged_at = NOW()
  LOG event ACKNOWLEDGMENT_SENT
  
  IF claim.status = PENDING_ACKNOWLEDGMENT
    THEN:
      TRANSITION status to COMPLETED
      SET claim.completed_at = NOW()
```

### 7.9 SLA Monitoring

```
REQUIREMENT 7.9.1: SLA At-Risk Alert
EVERY 5 minutes:
  at_risk = SELECT FROM claims WHERE:
    status NOT IN [COMPLETED, FAILED]
    AND acknowledged_at IS NULL
    AND (NOW() - received_at) > 90 minutes
    AND NOT EXISTS escalation with reason_code = SLA_AT_RISK for this claim
  
  FOR each claim IN at_risk:
    CREATE Escalation with reason_code = SLA_AT_RISK, priority = URGENT
    LOG event ESCALATION_CREATED
    NOTIFY supervisor
```

```
REQUIREMENT 7.9.2: SLA Breach Logging
EVERY 5 minutes:
  breached = SELECT FROM claims WHERE:
    acknowledged_at IS NULL
    AND (NOW() - received_at) > 120 minutes
  
  FOR each claim IN breached:
    LOG event SLA_BREACH with claim details
    INCREMENT metrics.sla_breach_count
    NOTIFY supervisor with priority = CRITICAL
```

---

## 8. Integration Contracts

### 8.1 Policy Administration System (SOAP)

**[SCOPE-OUT: SOAP Schema Unknown]**

The exact SOAP request/response schema is not provided in the scenario. The following is a logical contract based on typical policy administration systems. **This must be validated with client IT before implementation.**

**Resolution Plan**:
1. Request WSDL from client IT
2. Obtain test environment credentials
3. Validate assumed schema against actual
4. Update this contract with actual schema

**Assumed Contract** (to be validated):

**Endpoint**: `[SCOPE-OUT: URL to be provided by client IT]`

**Authentication**: `[SCOPE-OUT: Assumed WS-Security UsernameToken; validate with client]`

**Operation**: `GetPolicyByNumber`

**Request (assumed)**:
```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <GetPolicyByNumber xmlns="[SCOPE-OUT: namespace TBD]">
      <PolicyNumber>{policy_number}</PolicyNumber>
    </GetPolicyByNumber>
  </soap:Body>
</soap:Envelope>
```

**Response (assumed success)**:
```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <GetPolicyByNumberResponse>
      <Policy>
        <PolicyNumber>string</PolicyNumber>
        <Status>ACTIVE|LAPSED|CANCELLED|PENDING</Status>
        <PolicyholderName>string</PolicyholderName>
        <EffectiveDate>YYYY-MM-DD</EffectiveDate>
        <ExpirationDate>YYYY-MM-DD</ExpirationDate>
        <PolicyType>AUTO|HOMEOWNERS|COMMERCIAL|LIABILITY|WORKERS_COMP</PolicyType>
        <CoverageLimits>
          <Limit Type="string" Amount="decimal"/>
        </CoverageLimits>
        <Deductible>decimal</Deductible>
        <Exclusions>
          <Exclusion Code="string" Description="string"/>
        </Exclusions>
      </Policy>
    </GetPolicyByNumberResponse>
  </soap:Body>
</soap:Envelope>
```

**Response (not found)**:
```xml
<soap:Fault>
  <faultcode>POLICY_NOT_FOUND</faultcode>
  <faultstring>No policy found with number {policy_number}</faultstring>
</soap:Fault>
```

**Timeout and Retry**:
| Setting | Value |
|---------|-------|
| Connection timeout | 5 seconds |
| Read timeout | 15 seconds |
| Retry count | 2 |
| Retry backoff | Exponential: 2s, 4s |
| Circuit breaker threshold | 5 consecutive failures |
| Circuit breaker reset | 60 seconds |

**Rate Limit**: `[SCOPE-OUT: Unknown; assumed no limit. Validate with client.]`

**Fallback**: If circuit open or all retries exhausted → escalate claim with SYSTEM_ERROR; alert IT operations.

---

### 8.2 CRM System (API)

**[SCOPE-OUT: API Type Unknown]**

The scenario states "modern CRM with APIs" but does not specify REST vs. GraphQL vs. other. **Assumption A13: REST API. Must validate with client.**

**Resolution Plan**:
1. Confirm API type with client IT
2. Obtain API documentation
3. Obtain test environment credentials
4. Update this contract with actual schema

**Assumed Contract** (REST):

**Base URL**: `[SCOPE-OUT: URL to be provided by client IT]`

**Authentication**: `[SCOPE-OUT: Assumed OAuth 2.0 client credentials; validate with client]`

#### 8.2.1 Create Claim

**Endpoint**: `POST /api/v1/claims`

**Request**:
```json
{
  "claim_id": "uuid",
  "external_reference": "string|null",
  "channel": "EMAIL|PHONE_TRANSCRIPT|WEB_FORM",
  "received_at": "ISO8601 timestamp",
  "policy_number": "string",
  "claimant_name": "string",
  "claimant_email": "string|null",
  "claimant_phone": "string|null",
  "date_of_loss": "YYYY-MM-DD",
  "loss_description": "string",
  "loss_location": "string|null",
  "injury_indicated": "boolean",
  "fatality_indicated": "boolean",
  "estimated_damage": "decimal|null",
  "third_parties_involved": "boolean",
  "severity": "CRITICAL|STANDARD|LOW",
  "category": "AUTO|PROPERTY|LIABILITY|WORKERS_COMP|OTHER",
  "coverage_status": "COVERED|NOT_COVERED|PARTIAL|LAPSED|REVIEW_REQUIRED",
  "status": "string"
}
```

**Response (201 Created)**:
```json
{
  "crm_claim_id": "string",
  "created_at": "ISO8601 timestamp"
}
```

**Response (400 Bad Request)**:
```json
{
  "error": "string",
  "field": "string|null",
  "message": "string"
}
```

#### 8.2.2 Assign Claim to Adjuster

**Endpoint**: `PUT /api/v1/claims/{claim_id}/assign`

**Request**:
```json
{
  "adjuster_id": "uuid"
}
```

**Response (200 OK)**:
```json
{
  "assigned_at": "ISO8601 timestamp"
}
```

#### 8.2.3 Send Acknowledgment Email

**Endpoint**: `POST /api/v1/communications/email`

**Request**:
```json
{
  "to": "string (email address)",
  "template_id": "fnol_acknowledgment",
  "variables": {
    "claimant_name": "string",
    "claim_id": "string",
    "received_date": "string",
    "next_steps": "string",
    "contact_phone": "string",
    "contact_hours": "string"
  },
  "claim_id": "uuid"
}
```

**Response (202 Accepted)**:
```json
{
  "message_id": "string",
  "queued_at": "ISO8601 timestamp"
}
```

**Timeout and Retry**:
| Setting | Value |
|---------|-------|
| Connection timeout | 5 seconds |
| Read timeout | 10 seconds |
| Retry count | 3 |
| Retry backoff | Exponential: 1s, 2s, 4s |

**Rate Limit**: `[SCOPE-OUT: Unknown; validate with client]`

**Fallback**: If all retries fail → log error; claim remains in current state; alert IT operations.

---

### 8.3 Document Management System

**[SCOPE-OUT: Deferred to v2]**

Document attachment handling (storing email attachments, extracting content from documents) is out of scope for v1.

**v1 Behavior**: Email attachments are stored as raw files via DMS API; no content extraction. Attachment metadata linked to claim record.

**Resolution Plan for v2**:
1. Obtain DMS API documentation
2. Define document classification requirements
3. Evaluate OCR/extraction needs

---

## 9. Error Handling

### 9.1 Missing Data

| Scenario | Agent Behavior |
|----------|----------------|
| policy_number not extractable | Retry extraction once; if still < 0.90 confidence, escalate with LOW_EXTRACTION_CONFIDENCE |
| claimant_email not extractable | Proceed; skip acknowledgment email; log warning; claim continues processing |
| date_of_loss not extractable | Retry extraction once; if still < 0.90 confidence, escalate with LOW_EXTRACTION_CONFIDENCE |
| Non-critical field not extractable | Proceed; mark field as LOW_CONFIDENCE in claim record |

### 9.2 System Failures

| System | Failure Mode | Agent Behavior |
|--------|--------------|----------------|
| Email inbox | Connection failed | Retry 3x with 5s backoff; if still failing, alert IT; polling pauses |
| Policy Admin (SOAP) | Timeout | Retry 2x with exponential backoff (2s, 4s); if still failing, escalate claim with SYSTEM_ERROR |
| Policy Admin (SOAP) | 5+ consecutive failures | Open circuit breaker; escalate all pending claims; alert IT; retry after 60s |
| CRM (create claim) | Failure | Retry 3x with backoff; if failing, log locally; alert IT; do not lose claim data |
| CRM (send email) | Failure | Retry 3x; if failing, claim continues but acknowledged_at remains NULL; alert IT |

### 9.3 Conflict Resolution

| Conflict | Resolution |
|----------|------------|
| Multiple policies found | Escalate with all policy numbers in escalation detail; human selects |
| Extracted date_of_loss in future | Flag as anomaly; set date_of_loss_confidence = 0.50; escalate |
| Estimated damage negative | Set to NULL; proceed; add note to claim record |
| Human override contradicts agent | Human decision takes precedence; log override with human_id and rationale |

---

## 10. Audit and Logging

### 10.1 Required Events

Every processing step must log an event with: log_id, timestamp, claim_id, agent_id, event_type, relevant data.

### 10.2 Retention

| Log Type | Retention Period |
|----------|------------------|
| Claim processing events | 7 years (insurance regulation assumption) |
| System health metrics | 1 year |
| Debug/trace logs | 30 days |

### 10.3 Compliance

- All coverage validation decisions traceable to source policy data
- All escalation decisions traceable to triggering condition
- Human override actions must capture: user_id, timestamp, original_decision, override_decision, rationale
- PII masked in debug logs (claimant name, email, phone redacted)

---

## 11. Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `extraction.confidence.critical_threshold` | 0.90 | Minimum confidence for critical fields |
| `extraction.confidence.noncritical_threshold` | 0.70 | Minimum confidence for non-critical fields |
| `extraction.retry.max_attempts` | 1 | Retries before escalation |
| `triage.severity.critical_damage_threshold` | 100000 | USD threshold for CRITICAL |
| `triage.severity.standard_damage_threshold` | 25000 | USD threshold for STANDARD |
| `triage.injury_confidence_threshold` | 0.80 | Below this, escalate injury claims |
| `routing.high_value.damage_threshold` | 50000 | USD threshold for high-value escalation |
| `routing.high_value.policy_limit_threshold` | 500000 | USD threshold for high-value escalation |
| `routing.category.confidence_threshold` | 0.85 | Minimum for automated routing |
| `sla.acknowledgment.warning_minutes` | 90 | Minutes before SLA at-risk alert |
| `sla.acknowledgment.breach_minutes` | 120 | SLA breach threshold |
| `policy_lookup.timeout_seconds` | 15 | SOAP call timeout |
| `policy_lookup.retry_count` | 2 | Retry attempts |
| `policy_lookup.circuit_breaker.failure_threshold` | 5 | Failures to open circuit |
| `policy_lookup.circuit_breaker.reset_seconds` | 60 | Time to half-open |

---

## 12. Non-Functional Requirements

| Requirement | Target | Measurement |
|-------------|--------|-------------|
| Processing latency (P50) | < 2 minutes | Intake to acknowledgment |
| Processing latency (P95) | < 5 minutes | Intake to acknowledgment |
| System availability | >= 99.5% | Monthly uptime |
| Throughput | 300+ claims/day | Daily volume capacity |
| Burst capacity | 50 claims/hour | Peak handling |
| Extraction accuracy | >= 95% field-level | Weekly sample audit |
| False escalation rate | <= 10% | Weekly sample audit |
