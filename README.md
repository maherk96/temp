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
public class CoverageSpecUpsertRequest {

  @NotNull
  private Integer schemaVersion;

  @NotNull
  @Valid
  private Spec spec;

  @NotEmpty
  @Valid
  private List<Pillar> pillars;

  /* =========================
     Spec
     ========================= */

  @Data
  @JsonInclude(JsonInclude.Include.NON_NULL)
  public static class Spec {

    @NotBlank
    private String applicationKey;

    @NotBlank
    private String name;

    @NotNull
    private SpecStatus status; // DRAFT / PUBLISHED / ARCHIVED

    private String description;

    @Valid
    private DefaultPolicies defaultPolicies;
  }

  @Data
  @JsonInclude(JsonInclude.Include.NON_NULL)
  public static class DefaultPolicies {
    /** If an item allows MANUAL but doesn't specify expiryDays, use this default. */
    private Integer defaultManualEvidenceExpiryDays;

    /** If an item requires AUTOMATED but doesn't specify evidenceWindowDays, use this default. */
    private Integer defaultEvidenceWindowDaysForAutomation;
  }

  public enum SpecStatus {
    DRAFT, PUBLISHED, ARCHIVED
  }

  /* =========================
     Pillar / Capability / Item
     ========================= */

  @Data
  @JsonInclude(JsonInclude.Include.NON_NULL)
  public static class Pillar {

    @NotBlank
    private String pillarKey; // e.g. ORD, CTRL

    @NotBlank
    private String name;

    private String description;

    @NotEmpty
    @Valid
    private List<Capability> capabilities;
  }

  @Data
  @JsonInclude(JsonInclude.Include.NON_NULL)
  public static class Capability {

    @NotBlank
    private String capabilityKey; // e.g. ORD-CREATE

    @NotBlank
    private String name;

    private String description;

    @NotEmpty
    @Valid
    private List<CoverageItem> coverageItems;
  }

  @Data
  @JsonInclude(JsonInclude.Include.NON_NULL)
  public static class CoverageItem {

    @NotBlank
    private String coverageItemId; // stable ID like ORD-CANCEL-001

    @NotBlank
    private String title;

    private String description;

    @NotNull
    private RiskLevel risk;

    private List<String> tags;

    @Valid
    private Scope scope;

    private List<String> acceptanceCriteria;

    @NotNull
    @Valid
    private EvidencePolicy evidencePolicy;

    @NotNull
    @Valid
    private AutomationPolicy automationPolicy;
  }

  public enum RiskLevel {
    LOW, MEDIUM, HIGH
  }

  @Data
  @JsonInclude(JsonInclude.Include.NON_NULL)
  public static class Scope {
    private List<String> environments; // e.g. UAT, PROD-LIKE
    private List<String> components;   // e.g. OrderService
  }

  /* =========================
     Evidence
     ========================= */

  @Data
  @JsonInclude(JsonInclude.Include.NON_NULL)
  public static class EvidencePolicy {

    @NotEmpty
    private List<EvidenceType> allowedEvidence;

    /**
     * Usually equals allowedEvidence, but can be stricter.
     * Example: allowed [AUTO, MANUAL], requireAtLeastOneOf [AUTO]
     */
    @NotEmpty
    private List<EvidenceType> requireAtLeastOneOf;

    @Valid
    private ManualRules manualRules;

    @Valid
    private AutomationRules automationRules;
  }

  public enum EvidenceType {
    AUTOMATED, MANUAL
  }

  @Data
  @JsonInclude(JsonInclude.Include.NON_NULL)
  public static class ManualRules {
    /** Evidence expires after N days (attestation must be renewed). */
    private Integer expiryDays;

    private Boolean requiresApproval;

    /** e.g. QA_APPROVER, RISK_APPROVER, COMPLIANCE_OFFICER */
    private String requiredRole;

    private Boolean requiresAttachment;
  }

  @Data
  @JsonInclude(JsonInclude.Include.NON_NULL)
  public static class AutomationRules {
    /** Automated evidence must have a passing run within this many days. */
    private Integer evidenceWindowDays;

    /** If true, latest evidence must be a PASS (not just "ran"). */
    private Boolean requirePass;
  }

  /* =========================
     Automation expectation
     ========================= */

  @Data
  @JsonInclude(JsonInclude.Include.NON_NULL)
  public static class AutomationPolicy {

    @NotNull
    private AutomationExpectation expectation; // MUST / SHOULD / MANUAL_OK

    private String reason;
  }

  public enum AutomationExpectation {
    MUST_AUTOMATE,
    SHOULD_AUTOMATE,
    MANUAL_OK
  }
}

```
