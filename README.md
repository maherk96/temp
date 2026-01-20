You are working in a Spring Boot (Java) backend with JPA/Hibernate and an existing Oracle database schema (QAPORTAL). 
I already store test execution/reporting data (tests, test runs, launches, tags, etc.). 

Goal:
Implement support for Coverage Specifications using the following input DTO:
CoverageSpecUpsertRequest (schemaVersion, spec, pillars -> capabilities -> coverageItems, evidencePolicy, automationPolicy).
This data must be stored in the database and later used for coverage reporting and linking tests to coverage items.

Critical requirements:
1) FIRST: analyze my current database schema (I will paste my existing DDL). 
   - Identify whether any suitable tables already exist for storing coverage specs, items, pillars, etc.
   - Reuse existing tables when appropriate.
   - Only create new tables/columns where necessary.

2) DB design:
   - Propose the minimal set of new tables needed to store:
     - Coverage Spec (draft/published/archived, version, schemaVersion, appKey, name, description, created/updated)
     - Pillars
     - Capabilities
     - Coverage Items (atomic units)
     - Evidence Policy (allowed evidence types, manual rules, automation rules)
     - Automation Policy (MUST_AUTOMATE/SHOULD_AUTOMATE/MANUAL_OK, reason)
   - Include:
     - primary keys
     - foreign keys
     - unique constraints (e.g., coverageItemId unique per application/spec)
     - indexes for reporting queries (by appKey, pillarKey, coverageItemId, status)
     - a versioning strategy (published specs immutable; new version created on changes)
   - Provide the full Oracle DDL for any new tables, including constraints and indexes.

3) Service implementation:
   - Create a Spring service class (e.g., CoverageSpecService) with a method like:
       upsertCoverageSpec(CoverageSpecUpsertRequest request, String currentUser)
   - The service must:
     - Validate the request (schemaVersion supported, uniqueness of IDs, evidence policy consistency)
     - Upsert in DRAFT mode:
        - if spec exists in DRAFT for the app+name (or specId provided), update it
        - otherwise create a new spec
     - Persist the full nested structure:
        - pillars -> capabilities -> coverage items
        - remove or mark stale records that were deleted in the new request (use a safe approach: soft-delete or diff-based update)
     - Store default policies at spec-level and allow item-level overrides
     - Be transactional and safe for concurrent updates
     - Return a response DTO summarizing created/updated counts and the spec version/id

4) Do NOT generate controllers yet.
5) Do NOT generate code for coverage reporting or manual sessions yet.
6) Keep the design compatible with my existing test reporting system, since later we will add a link table to map coverage items to existing testIds (JUnit/Cucumber).

What I will provide:
- My existing Oracle DDL (paste it and use it as the source of truth)
- The CoverageSpecUpsertRequest DTO already exists in my codebase

What you must output:
A) A short analysis of the current schema showing what can be reused and what is missing
B) The proposed new/updated Oracle DDL (tables + constraints + indexes)
C) The Spring service class implementation (entities + repositories only if needed, but focus on the service logic)
D) Any validation rules you recommend enforcing at the service layer
E) A brief note on how you would handle spec publishing/versioning (no code, just clear rules)

Important:
- Assume Oracle DB
- Assume JPA + Spring @Transactional
- Prefer DTOs instead of returning entities
- Avoid accessing repositories directly from controllers; service layer owns operations

```java

package com.yourcompany.coverage.dto;

import com.fasterxml.jackson.annotation.JsonInclude;
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotEmpty;
import jakarta.validation.constraints.NotNull;
import lombok.Data;

import java.util.List;

@Data
@JsonInclude(JsonInclude.Include.NON_NULL)
public class CoverageItemTestLinkRequest {

  @NotBlank
  private String coverageItemId;

  @NotEmpty
  @Valid
  private List<TestLink> links;

  @Data
  @JsonInclude(JsonInclude.Include.NON_NULL)
  public static class TestLink {

    @NotNull
    private Long testId;

    @NotNull
    private TestType testType;

    // JUnit identity
    private String className;
    private String methodName;

    // Cucumber identity
    private String featureName;
    private String scenarioName;

    /**
     * Optional: store how this link was created:
     * UI_MANUAL, TAGGED, IMPORTED, INFERRED
     */
    private String linkType;
  }

  public enum TestType {
    JUNIT_METHOD,
    JUNIT_CLASS,
    CUCUMBER_SCENARIO,
    CUCUMBER_FEATURE
  }
}

```
