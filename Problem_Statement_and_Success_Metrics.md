# Deliverable 1: Problem Statement & Success Metrics

## Current Operational State

### Volume and Timing
- **Daily volume**: 300 FNOL reports per day
- **Input channels**: Unstructured text from email, phone transcripts, and web forms
- **Required SLA**: All claims must be triaged, validated, routed, and acknowledged within 2 hours of receipt
- **Current staffing**: 12 specialists
- **Average handling time**: 22 minutes per claim

### Capacity Analysis
- **Theoretical daily capacity**: 12 specialists × 8 hours × (60 min / 22 min per claim) = 262 claims/day at 100% utilization
- **Actual demand**: 300 claims/day
- **Capacity gap**: 38 claims/day (14.5% over capacity)
- **Implication**: Team is structurally under-resourced, explaining high SLA breach rate

### Current Performance Baseline
| Metric | Current State | Implication |
|--------|---------------|-------------|
| Routing error rate | 18% | 54 claims/day misrouted; requires rework, delays resolution |
| SLA breach rate | 31% | 93 claims/day exceed 2-hour window; regulatory/contractual risk |
| Handling time | 22 minutes | Leaves no buffer for complex claims or volume spikes |

---

## Problem Statement: Claimant Perspective

**Who is the claimant**: A person who has just experienced a loss—car accident, property damage, injury, theft. They are filing a first notice of loss (FNOL) to initiate the claims process.

**The claimant's state of mind**:
- **Anxiety**: They have just experienced an adverse event. They are stressed, possibly injured, and uncertain about what happens next.
- **Urgency**: They need to know their claim was received and will be handled. Silence creates doubt.
- **Confusion**: They may not understand the claims process. They don't know what information is needed, what the timeline is, or who to contact.

**Where friction occurs for the claimant**:

1. **Acknowledgment delay**: With a 31% SLA breach rate, nearly 1 in 3 claimants waits more than 2 hours to hear that their claim was received. During this time, they don't know if their submission went through, if they need to re-submit, or if anyone is working on their case.

2. **Routing errors create downstream delays**: When 18% of claims are misrouted, those claimants experience additional delay as their claim is transferred to the correct adjuster. They may receive conflicting communications or be asked to repeat information.

3. **No visibility into process**: Claimants submit unstructured information (email, phone call, web form) and receive no immediate feedback about what happens next. They are left waiting without context.

4. **Inconsistent experience**: Depending on channel (email vs. phone vs. web), time of day, and which specialist handles their claim, the claimant experience varies. There is no standardized first response.

**What claimants need**:
- Immediate confirmation that their claim was received
- A claim reference number they can use for follow-up
- Clear next steps (what will happen, when to expect contact)
- Assurance that someone is working on their case
- Accurate routing so the right person contacts them the first time

---

## Problem Statement: Specialist Perspective

**Who is doing the work**: 12 claims specialists manually processing FNOL reports from three unstructured input channels.

**Where friction occurs**:

1. **Input normalization burden**: Specialists must parse unstructured text from emails, phone transcripts, and web forms—each with different structures, quality levels, and completeness. No standardization occurs before claims reach specialists.

2. **Repetitive policy lookup**: Every claim requires manual lookup against the legacy policy administration system (SOAP endpoints) to validate coverage. This is mechanical work consuming specialist time.

3. **Routing decision ambiguity**: 18% error rate indicates either (a) routing rules are unclear, (b) information needed for routing is frequently incomplete, or (c) time pressure forces rushed decisions. Misroutes create rework and delays.

4. **Volume exceeds capacity**: At 22 minutes per claim with 300 daily claims, the 12-person team cannot meet demand within an 8-hour workday. Specialists are forced to triage what to prioritize, leading to SLA breaches.

5. **Acknowledgment bottleneck**: Acknowledging claimants is required within 2 hours but depends on completing prior steps. When volume spikes or complex claims occur, acknowledgments are delayed.

---

## Problem Statement: Business Perspective

### Cost of Current State

**Direct labor cost** (assumption A1: loaded cost of $75,000/year per specialist):
- 12 specialists × $75,000 = $900,000/year baseline

**Cost of routing errors**:
- 54 misrouted claims/day × 250 workdays = 13,500 errors/year
- If each error requires 15 minutes of rework (assumption A3): 3,375 hours/year = 1.6 FTE equivalent
- Rework cost: ~$120,000/year

**Cost of SLA breaches** (assumption A2: $50 per breach):
- 93 breaches/day × 250 workdays = 23,250 breaches/year
- At $50 per breach: $1,162,500/year exposure
- Note: Actual penalty structure unknown; must validate with client

**Opportunity cost**:
- Specialists spend 22 minutes on routine claims that could be handled in <5 minutes with automation
- High-value specialist judgment is diluted by mechanical tasks

### Scalability Constraint
Current model scales linearly: +10% volume requires +10% headcount. With 300 claims/day already exceeding capacity, any growth in policy volume or claim frequency will degrade performance further.

### Risk Exposure
- **Compliance risk**: Depending on jurisdiction, FNOL processing SLAs may be regulatory requirements. 31% breach rate creates audit exposure (unknown U6: regulatory requirements must be validated).
- **Customer experience risk**: Delayed acknowledgments and misrouted claims create friction at moment of highest customer anxiety.
- **Operational fragility**: No buffer for volume spikes, staff illness, or complex claim surges.

---

## Success Metrics

### Primary Metrics (Tied to Scenario Numbers)

| Metric | Current Baseline | Target | Measurement Method |
|--------|------------------|--------|-------------------|
| **SLA compliance rate** | 69% (100% - 31% breach) | ≥95% | (Claims acknowledged within 2h) / (Total claims received) measured daily |
| **Routing accuracy** | 82% (100% - 18% error) | ≥95% | (Claims routed to correct adjuster on first attempt) / (Total claims routed) measured weekly via adjuster feedback |
| **Average handling time (automated path)** | 22 minutes | ≤3 minutes | Timestamp delta: claim receipt → acknowledgment sent, for claims not escalated to human |
| **Human escalation rate** | N/A (100% human today) | 15-25% | (Claims requiring human review) / (Total claims) measured daily |
| **End-to-end automation rate** | 0% | ≥75% | (Claims processed without human intervention) / (Total claims) measured daily |

### Secondary Metrics (Operational Health)

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| Extraction accuracy (field-level) | ≥95% | Sample 50 claims/week; compare agent extraction to human extraction |
| Coverage validation accuracy | ≥98% | Sample 50 claims/week; compare agent determination to specialist determination |
| False escalation rate | ≤10% | (Escalated claims that did not require human judgment) / (Escalated claims) via specialist feedback |
| System availability | ≥99.5% | Uptime monitoring of agent pipeline |
| P95 processing latency | ≤5 minutes | 95th percentile of claim processing time |

### Claimant-Centric Metrics

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| Time to first acknowledgment | ≤5 minutes (automated path) | Timestamp delta: claim receipt → acknowledgment sent |
| Acknowledgment completeness | 100% contain claim number + next steps | Audit of acknowledgment content |
| Claimant re-contact rate | ≤5% | (Claimants who call/email asking "did you get my claim?") / (Total claims) |

### Leading Indicators
- Extraction confidence score distribution (should trend higher over time)
- Escalation queue depth (should remain stable, not grow)
- Policy lookup latency from SOAP system (impacts end-to-end time)
- Error rate by input channel (identifies channel-specific issues)

### Lagging Indicators
- Adjuster satisfaction with claim quality (quarterly survey)
- Claimant NPS related to initial contact (quarterly survey)
- Regulatory audit findings related to FNOL handling (annual)
- Specialist overtime hours (should decrease)

---

## Investment Justification

### Conservative ROI Model

**Assumptions**:
- 75% of claims automated end-to-end
- Remaining 25% still require specialist review but with pre-populated data (saving 50% of handling time)
- Routing accuracy improvement eliminates 80% of rework
- SLA compliance improvement avoids regulatory penalties (value: $50 per breach avoided—assumption A2, flagged for validation)

**Annual Savings**:
| Category | Calculation | Annual Value |
|----------|-------------|--------------|
| Specialist time freed (automated claims) | 225 claims/day × 22 min × 250 days = 20,625 hours | ~$750,000 in redeployable capacity |
| Rework reduction | 10,800 errors avoided × 15 min = 2,700 hours | ~$100,000 |
| SLA breach penalties avoided | 18,600 breaches avoided × $50 | $930,000 |
| **Total annual value** | | **~$1.78M** |

**Investment required**: Platform build + integration + first-year operations estimated at $300,000-$500,000.

**Payback period**: <6 months at conservative estimates.

**Sensitivity analysis**: If SLA breach cost is $0 (no regulatory penalty), savings reduce to ~$850,000—still positive ROI within 12 months.

---

## Constraints and Risks

### Hard Constraints
1. **2-hour SLA is non-negotiable**: System must acknowledge 100% of claims within this window or escalate with human notification.
2. **Human oversight for high-value/ambiguous claims**: Client explicitly requires this; full automation is not acceptable for these cases.
3. **Integration with existing systems**: Must work with CRM (APIs—type unspecified, assumed REST per A13), policy admin (SOAP), and document management—no system replacement.
4. **Legacy SOAP dependency**: Policy administration system uses SOAP endpoints; this is stated in the scenario and is a hard constraint.

### Risks to Mitigate
| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Legacy SOAP system latency causes SLA breaches | Medium | High | Circuit breaker pattern; escalate if lookup exceeds 30 seconds |
| Extraction accuracy insufficient for automation | Medium | High | Conservative confidence thresholds; specialist backup |
| Client's "ambiguous" definition broader than assumed | Medium | Medium | Start conservative; tune thresholds based on feedback |
| Specialists resist new workflow | Low | Medium | Involve specialists in UAT; position as augmentation |
| Regulatory requirements stricter than assumed | Low | High | Early compliance review; build flexibility into acknowledgment |
