```java
private List<HeatmapAnalysisDTO> buildHeatmapResponse(
        List<TestTagStatisticsData> data,
        LocalDateTime userStartDate
) {
    DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd MMM yyyy");

    return data.stream()
        .map(row -> new HeatmapAnalysisDTO(
                row.getClassName(),
                row.getTagName(),
                row.getPassedTests(),
                row.getPassRate(),
                row.getTotalTests(),

                // ACTIVE if periodIdx == 0, else STALE
                row.getPeriodIdx() == 0 ? HeatmapStatus.ACTIVE : HeatmapStatus.STALE,

                // format latest run
                row.getLatestRunTime() != null
                        ? row.getLatestRunTime().format(fmt)
                        : "N/A",

                // period description
                row.getPeriodAnalysis()
        ))
        .toList(); // Java 16+
}

public HeatmapDetailsDTO getHeatmapById(
        Long id,
        LocalDateTime startDate,
        LocalDateTime endDate,
        boolean regression
) {
    try {
        var tagIds = getTagIds(id);
        var appId = getAppId(id);

        List<TestTagStatisticsData> testTagData =
                getHeatmapAnalysisData(appId, startDate, endDate, tagIds.toArray(new Long[0]), regression);

        var heatmap = heatmapDomainHelper.findHeatmapEntityById(id);

        var username = heatmap.getUser() != null
                ? heatmap.getUser().getName()
                : "Unknown User";

        HeatmapResponseDTO header = new HeatmapResponseDTO(
                heatmap.getId(),
                username,
                heatmap.getName(),
                heatmap.getDescription(),
                heatmap.getDeleted(),
                heatmap.getApp().getId(),
                tagIds
        );

        return new HeatmapDetailsDTO(
                header,
                buildHeatmapResponse(testTagData, startDate)
        );

    } catch (Exception e) {
        log.error("Error generating weekly tag regression for heatmap {}: {}", id, e.getMessage(), e);
        throw new IllegalStateException(e);
    }
}
```
