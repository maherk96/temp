```java
@GetMapping
@Operation(
    summary = "Get paginated dashboards",
    description = "Returns a paginated list of dashboards with optional search, type, and deleted filters."
)
public ResponseEntity<PaginatedResponse<DashboardDTO>> getDashboards(
        @RequestParam(defaultValue = "1") int page,
        @RequestParam(defaultValue = "10") int size,
        @RequestParam(required = false) String search,
        @RequestParam(required = false) String dashboardType,
        @RequestParam(defaultValue = "false") boolean deleted) {

    String currentUser = "mk66440"; // TODO: replace with authenticated user

    Page<DashboardDTO> dashboardPage = dashboardManagementService.findAll(
            search, dashboardType, currentUser, deleted, PageRequest.of(page - 1, size));

    return ResponseEntity.ok(PaginatedResponseMapper.from(dashboardPage));
}
```
