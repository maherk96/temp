```java
@Test
@DisplayName("Test Coverage Metrics Data with correct weekly windowing")
void testCoverageMetricsDataWithWeeklyWindows() {

    // Align everything to week boundaries
    LocalDateTime now = LocalDateTime.now();
    LocalDate weekStart = now.toLocalDate()
                             .with(java.time.DayOfWeek.MONDAY);
    
    LocalDateTime p0Start = weekStart.atStartOfDay();       // This week
    LocalDateTime p0End   = p0Start.plusWeeks(1);

    LocalDateTime p1Start = p0Start.minusWeeks(1);          // Last week
    LocalDateTime p1End   = p0Start;

    LocalDateTime p2Start = p0Start.minusWeeks(2);          // 2 weeks ago
    LocalDateTime p2End   = p1Start;

    LocalDateTime p3Start = p0Start.minusWeeks(3);          // 3 weeks ago
    LocalDateTime p3End   = p2Start;

    List<TestTagStatisticsData> rawData = new ArrayList<>();

    // Period 0: This week (fresh)
    rawData.add(new TestTagStatisticsData(
            "OrderStates", "COMMON", 0,
            p0Start, p0End,
            20, 19, 3,
            p0Start.plusDays(1)
    ));

    // Period 1: Last week
    rawData.add(new TestTagStatisticsData(
            "OrderStates", "COMMON", 1,
            p1Start, p1End,
            20, 15, 3,
            p1Start.plusDays(2)
    ));

    // Period 2: Two weeks ago
    rawData.add(new TestTagStatisticsData(
            "OrderStates", "COMMON", 2,
            p2Start, p2End,
            20, 14, 3,
            p2Start.plusDays(3)
    ));

    // Period 3: Three weeks ago
    rawData.add(new TestTagStatisticsData(
            "OrderStates", "COMMON", 3,
            p3Start, p3End,
            20, 6, 3,
            p3Start.plusDays(4)
    ));

    // Execute
    heatmapDataService.buildHeapmapResponse(rawData, now);
}

```
