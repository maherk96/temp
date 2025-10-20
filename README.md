```java
@Query("SELECT w FROM DashboardWidget w WHERE w.dashboard.id = :dashboardId ORDER BY w.ordinal ASC")
List<DashboardWidget> findOrderedWidgetsByDashboardId(@Param("dashboardId") Long dashboardId);
```
