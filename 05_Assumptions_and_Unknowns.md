# Deliverable 5: Assumptions & Unknowns

## Overview

This document separates **assumptions** (things taken as given due to missing information) from **unknowns** (things that must be validated with the client before building). Assumptions are reasonable inferences; unknowns are blockers or significant risks.

All assumptions are scenario-relevant and testable. Generic placeholders ("we assume good data quality") are excluded.

---

## Assumptions

### A1: Specialist Loaded Cost

**Assumption**: Claims specialist loaded cost (salary + benefits + overhead) is approximately $75,000/year.

**Basis**: Industry average for insurance claims processors in US markets.

**Used for**: ROI calculations in problem statement.

**Testable claim**: If actual loaded cost is between $60,000 and $90,000, ROI remains positive.

**Impact if wrong**:
- If higher ($90K): ROI improves
- If lower ($50K): ROI decreases but remains positive if SLA breach value is accurate

**Confidence**: Medium

**Validation method**: Request headcount cost data from client HR/Finance.

---

### A2: SLA Breach Cost

**Assumption**: SLA breaches carry a cost of approximately $50 per incident.

**Basis**: Placeholder for regulatory penalties and customer satisfaction impact. Many insurance regulations impose penalties for delayed claim acknowledgment.

**Used for**: ROI calculations; justifying SLA compliance target.

**Testable claim**: There is a quantifiable cost (regulatory, contractual, or reputational) associated with SLA breaches.

**Impact if wrong**:
- If higher: Business case strengthens
- If lower or zero: Business case relies on efficiency gains (~$850K) rather than penalty avoidance (~$930K)
- If SLA is purely internal metric with no penalty: Investment justification changes significantly

**Confidence**: Low

**Validation method**: Request regulatory requirements and contractual SLA terms from client compliance/legal.

---

### A3: Routing Error Rework Time

**Assumption**: Each routing error requires approximately 15 minutes of rework to correct.

**Basis**: Reasonable estimate for administrative handoff, communication, and record update.

**Used for**: Quantifying cost of 18% routing error rate.

**Testable claim**: Rework time is measurable; actual range is likely 10-30 minutes.

**Impact if wrong**:
- If higher: Routing accuracy improvement has greater value
- If lower: Routing accuracy improvement has less value but remains positive

**Confidence**: Medium

**Validation method**: Ask client: "When a claim is misrouted, what is involved in correcting it? How long does it take?"

---

### A4: Routing Error Root Cause

**Assumption**: The 18% routing error rate is caused by some combination of: (a) unclear routing rules, (b) incomplete information at routing time, (c) time pressure forcing rushed decisions.

**Basis**: Common causes in high-volume manual processing.

**Used for**: Justifying agent-based routing improvement.

**Testable claim**: Root causes are identifiable and addressable by automation.

**Impact if wrong**: If root cause is external (e.g., stale adjuster data, adjuster unavailability), solution design may need additional components.

**Confidence**: Medium

**Validation method**: Ask client: "What do you believe causes routing errors? Do you track why claims get re-routed?"

---

### A5: High-Value Threshold Definition

**Assumption**: "High-value" claims requiring human oversight are defined as: estimated_damage > $50,000 OR policy_limit > $500,000.

**Basis**: Industry-typical thresholds. Client stated "high-value claims" require oversight but did not define the threshold.

**Used for**: Escalation trigger in delegation analysis and agent specification.

**Testable claim**: Threshold is verifiable; client can confirm or provide alternative.

**Impact if wrong**:
- If threshold is lower ($25K): More escalations; automation rate decreases
- If threshold is higher ($100K): Fewer escalations; may not satisfy client's risk tolerance

**Confidence**: Low

**Validation method**: Ask client: "At what claim value do you require mandatory human review?"

---

### A6: Adjuster Routing Factors

**Assumption**: Adjusters are assigned claims based on: (1) claim category and (2) geographic territory.

**Basis**: Common insurance operations model.

**Used for**: Adjuster selection algorithm in agent specification.

**Testable claim**: These are the primary routing factors; additional factors can be incorporated.

**Impact if wrong**: If routing also depends on adjuster skill level, customer tier, workload caps, or other factors, routing logic becomes more complex.

**Confidence**: Medium

**Validation method**: Ask client: "What factors determine which adjuster receives a claim? Is it purely category + territory, or are there other rules?"

---

### A7: Exclusion Matching Logic

**Assumption**: Coverage exclusions in the policy admin system are represented as structured codes that can be matched programmatically against loss description keywords.

**Basis**: Assumes exclusion data is structured (e.g., "THEFT", "FLOOD", "INTENTIONAL_DAMAGE").

**Used for**: Coverage validation logic.

**Testable claim**: Exclusions are enumerable; matching logic is codifiable.

**Impact if wrong**: If exclusions are free-text or require interpretation, coverage validation becomes human-led for more cases.

**Confidence**: Medium

**Validation method**: Request sample policy records showing how exclusions are stored.

---

### A8: Email Acknowledgment Sufficiency

**Assumption**: Email is an acceptable acknowledgment channel for all claimants.

**Basis**: Email address is extractable from most FNOL sources.

**Used for**: Acknowledgment output specification.

**Testable claim**: Claimants expect email acknowledgment; no regulatory requirement for other channels.

**Impact if wrong**: If some claimants require SMS, phone callback, or portal notification, additional output channels are needed.

**Confidence**: Medium

**Validation method**: Ask client: "How do claimants expect to receive acknowledgment? Are there segments without email access?"

---

### A9: No Labeled Training Data

**Assumption**: The client has no labeled historical data suitable for training ML models for extraction, classification, or routing.

**Basis**: Client stated "no AI infrastructure today."

**Used for**: Solution approach (rule-based + LLM extraction rather than supervised learning).

**Testable claim**: If labeled data exists, supervised models could improve accuracy.

**Impact if wrong**: If historical data exists with labeled fields, model training could improve extraction accuracy beyond baseline.

**Confidence**: Medium

**Validation method**: Ask client: "Do you have historical claims data with fields like severity, category, and outcome labeled by specialists?"

---

### A10: Audit Log Retention Period

**Assumption**: Audit logs must be retained for 7 years per insurance regulatory requirements.

**Basis**: Common US insurance regulation for claims records.

**Used for**: Logging system design.

**Testable claim**: Retention period is verifiable.

**Impact if wrong**: If retention is longer (10 years in some states) or shorter, storage architecture may need adjustment.

**Confidence**: High

**Validation method**: Confirm with client compliance team.

---

### A11: Severity Tiers

**Assumption**: Severity is classified into three tiers: CRITICAL, STANDARD, LOW.

**Basis**: Common FNOL triage model.

**Used for**: Triage logic in agent specification.

**Testable claim**: Client uses similar categorization.

**Impact if wrong**: If client uses 5-level severity or different definitions, triage rules need redesign.

**Confidence**: Medium

**Validation method**: Ask client: "How do you classify claim severity today? What are the categories?"

---

### A12: SOAP System Availability

**Assumption**: The legacy policy administration SOAP endpoint is available during business hours with response time under 15 seconds for typical queries.

**Basis**: Legacy systems often have performance constraints; 15 seconds is conservative.

**Used for**: Timeout configuration; SLA feasibility assessment.

**Testable claim**: Response time is measurable; availability windows are documented.

**Impact if wrong**:
- If slower (>30s): May need to decouple policy lookup from synchronous processing
- If limited availability (maintenance windows): SLA compliance affected during those periods

**Confidence**: Low

**Validation method**: Request performance SLA and maintenance schedule for policy admin system.

---

### A13: CRM API is REST

**Assumption**: The "modern CRM with APIs" mentioned in the scenario exposes REST APIs.

**Basis**: REST is most common for modern CRM systems. However, the scenario does not explicitly state REST vs. GraphQL vs. SOAP vs. other.

**Used for**: CRM integration contract design.

**Testable claim**: API type is verifiable.

**Impact if wrong**:
- If GraphQL: Request/response patterns change; queries replace endpoints
- If SOAP: XML handling required; similar to policy admin integration
- If proprietary: May require SDK or different integration approach

**Confidence**: Medium

**Validation method**: Confirm API type with client IT. Request API documentation.

---

## Unknowns (Must Be Validated Before Building)

### U1: SOAP API Contract (CRITICAL)

**Unknown**: The exact SOAP endpoint URL, WSDL, authentication mechanism, request/response schema, and error codes for the policy administration system.

**Why critical**: Cannot build policy lookup integration without actual contract.

**Blocked until resolved**: Yes—integration cannot be implemented.

**Validation actions**:
- [ ] Obtain WSDL from client IT
- [ ] Confirm authentication method (WS-Security, API key, mTLS, other)
- [ ] Obtain test environment access and credentials
- [ ] Execute sample queries to validate assumed schema
- [ ] Document actual error codes and handling

**Owner**: Client IT

**Resolution deadline**: Before development sprint 1

---

### U2: CRM API Contract (CRITICAL)

**Unknown**: The exact API endpoints, authentication method, request/response schemas, and rate limits for the CRM system.

**Why critical**: CRM is target for claim records, assignments, and email sends. Cannot build outputs without actual contract.

**Blocked until resolved**: Yes—all outputs require CRM integration.

**Validation actions**:
- [ ] Confirm API type (REST assumed; see A13)
- [ ] Obtain API documentation
- [ ] Confirm authentication mechanism (OAuth assumed)
- [ ] Obtain test environment access
- [ ] Test claim creation, adjuster assignment, email send
- [ ] Document rate limits if any

**Owner**: Client IT

**Resolution deadline**: Before development sprint 1

---

### U3: Definition of "Ambiguous" Claims (HIGH)

**Unknown**: What specifically makes a claim "ambiguous" in the client's operational definition? The scenario says human oversight is required for ambiguous claims but does not define ambiguity.

**Possible meanings**:
- Low extraction confidence?
- Conflicting information in the claim?
- Coverage edge cases?
- Claims that historically required specialist judgment?

**Why critical**: This defines a primary delegation boundary. Without clear definition, thresholds are arbitrary.

**Validation actions**:
- [ ] Interview claims specialists: "What makes a claim require your judgment vs. being routine?"
- [ ] Review sample of historically complex claims
- [ ] Document specific ambiguity criteria

**Owner**: Client Claims Operations

**Resolution deadline**: Before finalizing delegation thresholds

---

### U4: Extraction Field Requirements (HIGH)

**Unknown**: What specific fields must be extracted from FNOL submissions? The specification assumes standard fields, but client may have additional required fields or specific field definitions.

**Why critical**: Extraction is foundation; missing required fields creates compliance or operational gaps.

**Validation actions**:
- [ ] Obtain current FNOL intake form or checklist
- [ ] Identify required vs. optional fields
- [ ] Identify any client-specific fields (internal codes, special categories)
- [ ] Confirm field validation rules

**Owner**: Client Claims Operations

**Resolution deadline**: Before extraction model design

---

### U5: Volume Distribution by Channel (MEDIUM)

**Unknown**: What percentage of the 300 daily claims comes from each channel (email, phone transcript, web form)?

**Why critical**: Different channels have different extraction complexity. Phone transcripts are typically harder than structured web forms.

**Impact**: Affects accuracy expectations, testing priorities, and potential channel-specific handling.

**Validation actions**:
- [ ] Request volume breakdown by channel (last 3 months)
- [ ] Identify any channel-specific quality issues

**Owner**: Client Claims Operations

---

### U6: Regulatory Requirements for FNOL (HIGH)

**Unknown**: Are there state-specific or line-of-business-specific regulatory requirements for FNOL handling?

**Examples**: Some states require acknowledgment within 24 hours; some require specific content; some require multilingual support.

**Why critical**: Regulatory non-compliance creates legal and financial risk.

**Validation actions**:
- [ ] Identify states where policyholders are located
- [ ] Identify applicable regulations for FNOL handling
- [ ] Confirm acknowledgment timing requirements
- [ ] Confirm acknowledgment content requirements
- [ ] Confirm any language requirements

**Owner**: Client Compliance/Legal

**Resolution deadline**: Before acknowledgment template design

---

### U7: Adjuster Data and Territory Mapping (MEDIUM)

**Unknown**: How many adjusters are there? What are their specializations and territories? How is this data maintained?

**Why critical**: Adjuster selection requires this data. If stale or unavailable, routing fails.

**Validation actions**:
- [ ] Obtain adjuster roster with specializations and territories
- [ ] Determine where adjuster data is maintained (CRM, separate system?)
- [ ] Determine update frequency and process
- [ ] Confirm whether load balancing is required

**Owner**: Client Claims Operations

---

### U8: Historical Data for Validation (MEDIUM)

**Unknown**: Is there historical claim data available (anonymized) for testing extraction accuracy and routing accuracy?

**Why critical**: Without historical data, validation relies on synthetic tests. Historical data provides realistic accuracy baseline.

**Validation actions**:
- [ ] Request sample of 200+ historical FNOL submissions with outcomes
- [ ] Confirm data can be anonymized for testing
- [ ] Identify any data access restrictions

**Owner**: Client IT / Compliance

---

### U9: Specialist Role Post-Implementation (MEDIUM)

**Unknown**: What is the expected role of the 12 specialists after implementation? Pure escalation handling? Quality review? Other work?

**Why critical**: Change management and training depend on this. Specialist resistance can undermine adoption.

**Validation actions**:
- [ ] Clarify client's vision for specialist role post-automation
- [ ] Understand communication to the team
- [ ] Plan training for new workflow

**Owner**: Client Claims Management

---

### U10: Document Attachment Requirements (MEDIUM)

**Unknown**: Do FNOL submissions commonly include attachments? What types? Are attachments needed for initial triage?

**Why critical**: If attachments contain critical information (e.g., police report, photos), accuracy may be affected by not parsing them in v1.

**Validation actions**:
- [ ] What percentage of claims have attachments?
- [ ] What types of attachments (photos, documents, PDFs)?
- [ ] Are attachments needed for initial triage/routing, or only later?

**Owner**: Client Claims Operations

---

### U11: Acknowledgment Content Requirements (LOW)

**Unknown**: What exactly should the acknowledgment email contain? Branding, legal disclaimers, contact information specifics?

**Why critical**: Acknowledgment is customer-facing. Must meet brand and legal standards.

**Validation actions**:
- [ ] Obtain current acknowledgment template(s) if they exist
- [ ] Identify required elements (claim number, contact, disclaimers)
- [ ] Identify any variations by claim type or channel
- [ ] Compliance review of proposed content

**Owner**: Client Marketing / Compliance

---

### U12: Budget and Timeline (LOW for spec; HIGH for project)

**Unknown**: What is the client's budget and timeline expectation?

**Why critical**: Determines MVP scope and phasing.

**Validation actions**:
- [ ] Confirm budget range
- [ ] Confirm target go-live date
- [ ] Identify any external deadlines (regulatory, contractual)

**Owner**: Client Project Sponsor

---

## Summary: Validation Priority

| Priority | ID | Unknown | Blocker? |
|----------|----|---------|----------|
| **P0** | U1 | SOAP API Contract | Yes |
| **P0** | U2 | CRM API Contract | Yes |
| **P1** | U3 | "Ambiguous" Definition | Affects delegation |
| **P1** | U4 | Extraction Field Requirements | Affects core function |
| **P1** | U6 | Regulatory Requirements | Affects compliance |
| **P2** | U5 | Volume by Channel | Affects testing |
| **P2** | U7 | Adjuster Data | Affects routing |
| **P2** | U8 | Historical Data | Affects validation |
| **P2** | U9 | Specialist Role | Affects change management |
| **P2** | U10 | Attachment Requirements | Affects scope |
| **P3** | U11 | Acknowledgment Content | Can iterate |
| **P3** | U12 | Budget/Timeline | Project management |

---

## Explicit Scope-Outs with Resolution Plans

### Scope-Out 1: SOAP Request/Response Schema

**What is scoped out**: Exact XML structure for policy admin SOAP requests and responses.

**Why**: Not provided in scenario; assumed structure may differ from actual.

**Resolution plan**:
1. Client IT provides WSDL
2. Development team generates client code from WSDL
3. Schema in spec (Section 8.1) is updated to match actual

**Risk if not resolved**: Integration fails; development blocked.

---

### Scope-Out 2: CRM Endpoint URLs and Auth

**What is scoped out**: Actual endpoint URLs and authentication details for CRM.

**Why**: API documentation not provided.

**Resolution plan**:
1. Client IT provides API documentation
2. Development team configures endpoints and auth
3. Contract in spec (Section 8.2) is updated to match actual

**Risk if not resolved**: No outputs can be delivered.

---

### Scope-Out 3: Document Management System Integration

**What is scoped out**: Full DMS integration for attachment storage and retrieval.

**Why**: Deferred to v2; not required for core FNOL processing.

**v1 behavior**: Attachments stored as raw files; not parsed.

**Resolution plan for v2**:
1. Obtain DMS API documentation
2. Define document classification requirements
3. Evaluate OCR/extraction needs

**Risk if not resolved for v1**: Low—attachments stored but not analyzed.

---

## Risk Summary

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| SOAP system slower than assumed | Medium | High | Circuit breaker; async fallback; SLA at-risk escalation |
| Extraction accuracy below target | Medium | Medium | Conservative thresholds; human fallback |
| Client's "ambiguous" definition broader than modeled | Medium | Medium | Start conservative; tune thresholds with feedback |
| CRM API is not REST | Low | Medium | Adapt integration approach based on actual API type |
| Regulatory requirements stricter than assumed | Low | High | Early compliance review; flexible acknowledgment system |
| Specialists resist workflow change | Low | Medium | Involve in UAT; position as augmentation, not replacement |
