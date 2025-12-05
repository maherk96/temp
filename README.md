```java

public DashboardDatasetResponse getDashboardData(long dashboardId) {
    log.debug("Retrieving dashboard data for dashboardId: {}", dashboardId);

    List<DashboardWidgetData> rows;
    try {
        rows = queryExecutor.executeQuery(
                qapQueries.get(DASHBOARD_QUERY_KEY),
                new DashboardWidgetDataMapper(),
                dashboardId);
    } catch (DataAccessException e) {
        // Any problem talking to the DB / executing the query
        log.error("Database error while retrieving dashboard {}: {}", dashboardId, e.getMessage(), e);
        // Either rethrow DataAccessException or wrap it in your own service-level exception
        throw new DashboardDataAccessException(
                "Failed to retrieve dashboard data for id " + dashboardId, e);
    }

    if (rows == null || rows.isEmpty()) {
        log.warn("Dashboard not found: {}", dashboardId);
        throw new DashboardNotFoundException("Dashboard not found: " + dashboardId);
    }

    DashboardWidgetData firstRow = rows.get(0);
    DashboardDatasetResponse response = buildDashboardResponse(dashboardId, firstRow);

    List<WidgetResponse> widgets = rows.stream()
            .map(this::mapToWidgetResponse)
            .collect(Collectors.toList());

    response.setWidgetResponseList(widgets);

    log.info(
            "Successfully retrieved dashboard data for dashboardId: {} with {} widgets",
            dashboardId, widgets.size());

    return response;
}
```
