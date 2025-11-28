```java


public List<HeatmapAnalysisDTO> buildHeatmapResponse(
        List<TestTagStatisticsData> data,
        LocalDateTime userStartDate) {

    if (data == null || data.isEmpty()) {
        return Collections.emptyList();
    }

    Map<String, TestTagStatisticsData> mostRecentPerCell =
            data.stream()
                .collect(Collectors.toMap(
                    r -> r.getClassName() + "|" + r.getTagName(),
                    r -> r,
                    (a, b) -> a.getLatestRunTime().isAfter(b.getLatestRunTime()) ? a : b
                ));

    return mostRecentPerCell.values().stream()
            .map(row -> toAnalysisDto(row, userStartDate))
            .toList();
}
```
