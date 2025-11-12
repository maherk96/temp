```java
@GetMapping("/{id}")
@Operation(
    summary = "Weekly Regression Tag Analysis by Heatmap ID",
    description = "Provides % of non-failed tests per tag per class for a heatmap and date"
)
public ResponseEntity<List<HeatmapAnalysisData>> getWeeklyTagRegressionData(
        @PathVariable Long id,
        @RequestParam(defaultValue = "startDate") @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate startDate,
        @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate endDate,
        @RequestParam(defaultValue = "false") boolean regression
) {
    List<HeatmapAnalysisData> data = heatmapService.getHeatmapById(id, startDate, endDate, regression);
    return ResponseEntity.ok(data);
}

public List<HeatmapAnalysisData> getHeatmapById(
        Long id,
        LocalDate startDate,
        LocalDate endDate,
        boolean regression
) {
    try {
        List<Long> tagIds = getTagIds(id);
        Long appId = getAppId(id);

        List<TestTagData> testTagData =
                getAnalysisData(appId, startDate, endDate, tagIds, regression);

        return buildHeatmapResponse(testTagData, startDate);

    } catch (Exception e) {
        log.error("Error generating weekly tag regression for heatmap {}: {}", id, e.getMessage(), e);
        throw new IllegalStateException(e);
    }
}

private List<Long> getTagIds(Long heatmapId) {
    return heatmapDomainHelper.getTagIDsByHeatmapId(heatmapId).toList();
}

private Long getAppId(Long heatmapId) {
    return heatmapRepository
            .findById(heatmapId)
            .orElseThrow(() -> new IllegalArgumentException("Heatmap not found"))
            .getApp()
            .getId();
}

private HeatmapStatus determineStatus(LocalDate lastRun, LocalDate startDate) {
    if (lastRun == null) return HeatmapStatus.STALE;
    return lastRun.isAfter(startDate) ? HeatmapStatus.STALE : HeatmapStatus.ACTIVE;
}

private double calculatePassRate(int passed, int totalRuns) {
    if (totalRuns == 0) return 0.0;
    return (passed / (double) totalRuns) * 100;
}

private List<HeatmapAnalysisData> buildHeatmapResponse(
        List<TestTagData> input,
        LocalDate startDate
) {
    List<HeatmapAnalysisData> result = new ArrayList<>();

    for (var dataSet : input) {
        double passRate = calculatePassRate(dataSet.passed(), dataSet.totalTestRuns());
        LocalDate lastRun = parseDate(dataSet.lastRun());

        HeatmapStatus status = determineStatus(lastRun, startDate);

        HeatmapAnalysisData dto = new HeatmapAnalysisData(
                dataSet.testClassName(),
                dataSet.tag(),
                passRate,
                status,
                dataSet.totalTestRuns(),
                lastRun,
                dataSet.testRuns()
        );

        result.add(dto);
    }
    return result;
}

private LocalDate parseDate(String raw) {
    return raw == null ? null : LocalDate.parse(raw);
}
```
