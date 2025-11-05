```java

During application startup, Spring failed to initialize several beans due to a circular dependency between the following service classes:
HeatmapService
 ↳ HeatmapTagManagementService
    ↳ HeatmapTagService
       ↳ HeatmapManagementService
          ↳ HeatmapService

This created a dependency loop (A → B → C → D → A) where each service required another to be constructed first, preventing Spring from creating them at startup.

The main cause is tight coupling between the Heatmap-related services — each service directly depends on another for business logic that should be separated or shared via a neutral layer.
As a result:
	•	Each service violates single responsibility.
	•	There is no clear direction of dependency flow.
	•	It becomes difficult to test, maintain, or extend the code without creating further circular references.


To resolve the issue properly, we are restructuring the dependency graph to be acyclic and modular:
	1.	Introduce a shared domain helper/service
	•	Create a new class, e.g. HeatmapDomainHelper, containing the common reusable logic (such as fetching, linking, or validating heatmap–tag relationships).
	•	All existing services will depend on this helper instead of depending on each other.
	2.	Remove direct service-to-service dependencies
	•	Replace fields like private final HeatmapTagManagementService heatmapTagManagementService; with calls to the shared helper.
	•	This ensures that each service only manages its own domain logic.



```
