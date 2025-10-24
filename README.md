```java

public DashboardWidgetDetailsDto(
    Long id,
    OffsetDateTime created,
    String description,
    OffsetDateTime lastUpdated,
    String name,
    Integer ordinal,
    Long dashboardId,
    Long dashboardWidgetConfigId,
    String widgetConfiguration,
    String dashboardName,
    String dashboardDescription
) {
    this.id = id;
    this.created = created;
    this.description = description;
    this.lastUpdated = lastUpdated;
    this.name = name;
    this.ordinal = ordinal;
    this.dashboardId = dashboardId;
    this.dashboardWidgetConfigId = dashboardWidgetConfigId;
    this.widgetConfiguration = widgetConfiguration;
    this.dashboardName = dashboardName;
    this.dashboardDescription = dashboardDescription;
}

```
