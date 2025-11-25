```java
private List<HeatmapAnalysisDTO> buildHeatmapResponse(
        List<TestTagStatisticsData> data,
        LocalDateTime userStartDate
) {
    DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd MMM yyyy");

    // Filter – keep most recent period per class/tag
    List<TestTagStatisticsData> filtered =
            data.stream()
                .collect(Collectors.groupingBy(
                        row -> row.getClassName() + "|" + row.getTagName(),
                        Collectors.minBy(Comparator.comparing(TestTagStatisticsData::getPeriodIdx))
                ))
                .values()
                .stream()
                .flatMap(Optional::stream)
                .toList();

    return filtered.stream()
        .map(row -> {
            // derive ACTIVE/STALE from date, not index
            HeatmapStatus status;
            if (row.getLatestRunTime() == null) {
                status = HeatmapStatus.STALE;
            } else {
                status = row.getLatestRunTime().isBefore(userStartDate)
                        ? HeatmapStatus.STALE
                        : HeatmapStatus.ACTIVE;
            }

            String lastRun = row.getLatestRunTime() != null
                    ? row.getLatestRunTime().format(fmt)
                    : "N/A";

            return new HeatmapAnalysisDTO(
                    row.getClassName(),
                    row.getTagName(),
                    row.getPassedTests(),
                    row.getPassRate(),
                    row.getTotalTests(),
                    status,
                    lastRun,
                    row.getPeriodAnalysis(userStartDate)
            );
        })
        .toList();
}
```
