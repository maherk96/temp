Design a YAML-based Test Coverage Specification and explain how it should work end-to-end.

⸻

Requirements

1. Test Basis & Structure
	•	Coverage must be defined per Application → Pillar → Capability → Coverage Item
	•	A Coverage Item is the atomic unit used for coverage calculation
	•	Coverage Items must be uniquely identifiable and versionable
	•	YAML should be suitable for storage in Git and reviewed via PRs

2. Evidence Model
Coverage Items must support multiple forms of evidence:
	•	Automated evidence
	•	Linked to existing tests in the database
	•	Must support manual mapping using:
	•	DB test IDs (e.g. testCaseId)
	•	Stable test identifiers (e.g. class + method FQN)
	•	Manual evidence
	•	Represented via attestations or test sessions stored in DB
	•	Must support expiry (e.g. “valid for 30 days”)
	•	Must be auditable (who executed, when, environment, attachments)

The YAML must define what evidence is acceptable for each Coverage Item (automated, manual, or both).

3. Automation Expectations
	•	Each Coverage Item must declare an automation expectation, e.g.:
	•	MUST_AUTOMATE
	•	SHOULD_AUTOMATE
	•	MANUAL_OK
	•	This is used to:
	•	Compute automation coverage
	•	Highlight automation gaps
	•	Prevent penalising items that are intentionally manual

4. Manual Testing Considerations
	•	The system must NOT rely on ad-hoc prompts like “was this tested?”
	•	Manual testing must be recorded as structured, expirable evidence
	•	YAML should define the rules for manual evidence (expiry, required role, required attachments)

5. Reporting Goals
The design must enable generation of:
	•	% test coverage against the test basis
	•	% automation coverage of that test basis
	•	Identification of coverage gaps
	•	Identification of automation gaps
	•	Confidence level of reported coverage (based on evidence quality & freshness)

6. Practical Constraints
	•	Assume many legacy tests will never be updated with tags/annotations
	•	The design must provide immediate value using existing DB test identifiers
	•	Avoid fragile or high-maintenance solutions

⸻

Expected Output
	1.	A YAML schema example showing:
	•	Application, pillar, capability, coverage items
	•	Evidence policies
	•	Manual vs automated rules
	•	Explicit mapping to existing DB test IDs
	2.	An explanation of:
	•	How automated and manual evidence are resolved
	•	How coverage is calculated
	•	How automation gaps are identified
	3.	Recommended best practices to avoid data drift and stale mappings

