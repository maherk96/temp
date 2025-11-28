```java
public List<HeatmapAnalysisDTO> buildHeatmapResponse(
        List<TestTagStatisticsData> data,
        LocalDateTime userStartDate) {

    if (data == null || data.isEmpty()) {
        return Collections.emptyList();
    }

    // Group by className|tagName and pick the most recent entry
    Map<String, TestTagStatisticsData> mostRecentPerCell =
            data.stream()
                .collect(Collectors.toMap(
                    r -> r.getClassName() + "|" + r.getTagName(),
                    r -> r,
                    this::pickMoreRecent        // merge function
                ));

    // Map the selected rows to DTO
    return mostRecentPerCell.values().stream()
            .map(row -> toAnalysisDto(row, userStartDate))
            .toList();
}

private TestTagStatisticsData pickMoreRecent(
        TestTagStatisticsData a,
        TestTagStatisticsData b) {

    LocalDateTime ta = a.getLatestRunTime();
    LocalDateTime tb = b.getLatestRunTime();

    // If one is null, prefer the non-null one
    if (ta == null && tb != null) return b;
    if (tb == null && ta != null) return a;

    // If both are null, keep 'a' (deterministic)
    if (ta == null) return a;

    // Normal comparison
    return ta.isAfter(tb) ? a : b;
}

```
