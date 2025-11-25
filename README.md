```java

private List<HeatmapAnalysisDTO> buildHeatmapResponse(
        List<TestTagStatisticsData> data,
        LocalDateTime userStartDate
) {
    DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd MMM yyyy");

    // 1️⃣ Filter: For each className+tagName, keep only the LOWEST periodIdx
    List<TestTagStatisticsData> filtered =
            data.stream()
                .collect(Collectors.groupingBy(
                        row -> row.getClassName() + "|" + row.getTagName(),
                        Collectors.minBy(Comparator.comparing(TestTagStatisticsData::getPeriodIdx))
                ))
                .values()
                .stream()
                .map(Optional::get)
                .toList();

    // 2️⃣ Map to DTO
    return filtered.stream()
        .map(row -> new HeatmapAnalysisDTO(
                row.getClassName(),
                row.getTagName(),
                row.getPassedTests(),
                row.getPassRate(),
                row.getTotalTests(),

                // ACTIVE only if periodIdx = 0
                row.getPeriodIdx() == 0 ? HeatmapStatus.ACTIVE : HeatmapStatus.STALE,

                row.getLatestRunTime() != null
                        ? row.getLatestRunTime().format(fmt)
                        : "N/A",

                // Your nice user-friendly explanation
                row.getPeriodAnalysis(userStartDate)
        ))
        .toList();
}
```
