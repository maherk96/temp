```java
public DashboardDatasetResponse getDashboardData(long dashboardId) {
    var data = queryExecutor.executeQuery(
        qapQueries.get("DashboardDatasetQuery"),
        new DashboardWidgetDataMapper(),
        dashboardId
    );
    
    if (data.isEmpty()) {
        throw new DashboardNotFoundException("Dashboard not found: " + dashboardId);
    }
    
    var firstRow = data.getFirst();
    var response = new DashboardDatasetResponse();
    response.setId(dashboardId);
    response.setName(firstRow.getDashboardName());
    response.setDescription(firstRow.getDashboardDescription());
    
    List<WidgetResponse> widgets = data.stream()
        .map(this::mapToWidgetResponse)
        .toList();
    
    response.setWidgetResponseList(widgets);
    return response;
}

private WidgetResponse mapToWidgetResponse(DashboardWidgetData data) {
    try {
        var widgetConfig = JsonUtil.getObjectMapper()
            .readValue(data.getWidgetConfiguration(), WidgetConfig.class);
        
        var widgetResponse = new WidgetResponse();
        widgetResponse.setWidgetId(data.getDashboardWidgetConfigId());
        widgetResponse.setOrdinal(data.getOrdinal());
        widgetResponse.setName(data.getWidgetName());
        widgetResponse.setDescription(data.getWidgetDescription());
        widgetResponse.setWidgetDataSetType(widgetConfig.getType());
        widgetResponse.setWidgetConfig(widgetConfig);
        
        return widgetResponse;
    } catch (JsonProcessingException e) {
        throw new WidgetConfigurationException(
            "Failed to parse widget configuration for widget: " + data.getWidgetName(), e);
    }
}
```
