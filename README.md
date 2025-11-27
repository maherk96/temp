```java


public class HeatmapMetricsDTO {

    @HeatmapMetricsConfigField(
        label = "Freshness Score",
        icon = "refresh",
        value = 0
    )
    private double freshnessScore;

    @HeatmapMetricsConfigField(
        label = "Stale Tests",
        icon = "warning",
        value = 0
    )
    private long staleCount;

    @HeatmapMetricsConfigField(
        label = "Execution Coverage",
        icon = "coverage",
        value = 0
    )
    private double executionCoverage;

    @HeatmapMetricsConfigField(
        label = "Pass Rate",
        icon = "check",
        value = 0
    )
    private double passRate;

    @HeatmapMetricsConfigField(
        label = "Activity Trend",
        icon = "trend",
        value = 0
    )
    private double activityTrend; // 1=UP, 0=FLAT, -1=DOWN
}

public class HeatmapMetricsResponse {
    private List<HeatmapMetricItemDTO> metrics;

    public HeatmapMetricsResponse(List<HeatmapMetricItemDTO> metrics) {
        this.metrics = metrics;
    }
}


public class HeatmapMetricItemDTO {
    private String label;
    private String icon;
    private String value;

    public HeatmapMetricItemDTO(String label, String icon, String value) {
        this.label = label;
        this.icon = icon;
        this.value = value;
    }
}


private HeatmapMetricsResponse wrapMetrics(HeatmapMetricsDTO metricsDTO) {

    List<HeatmapMetricItemDTO> items = new ArrayList<>();

    for (Field field : metricsDTO.getClass().getDeclaredFields()) {
        field.setAccessible(true);

        HeatmapMetricsConfigField config = field.getAnnotation(HeatmapMetricsConfigField.class);
        if (config == null) continue;

        try {
            Object rawValue = field.get(metricsDTO);
            double numericValue = rawValue instanceof Number
                    ? ((Number) rawValue).doubleValue()
                    : 0;

            items.add(new HeatmapMetricItemDTO(
                config.label(),
                config.icon(),
                String.valueOf(numericValue)
            ));

        } catch (IllegalAccessException ignored) {}
    }

    return new HeatmapMetricsResponse(items);
}

private HeatmapMetricsDTO buildMetrics(List<TestTagStatisticsData> rawData,
                                       List<HeatmapAnalysisDTO> analysis) {

    HeatmapMetricsDTO dto = new HeatmapMetricsDTO();

    dto.setFreshnessScore(calcFreshnessScore(analysis));
    dto.setStaleCount(calcStaleCount(analysis));
    dto.setExecutionCoverage(calcExecutionCoverage(rawData));
    dto.setPassRate(calcPassRate(rawData));
    dto.setActivityTrend(calcActivityTrend(rawData));  // trend: +1, 0, -1

    return dto;
}


private double calcFreshnessScore(List<HeatmapAnalysisDTO> analysis) {
    long fresh = analysis.stream()
            .filter(a -> !"STALE".equals(a.getStatus()))
            .count();

    return analysis.isEmpty() ? 0 : (fresh * 100.0) / analysis.size();
}

private long calcStaleCount(List<HeatmapAnalysisDTO> analysis) {
    return analysis.stream()
            .filter(a -> "STALE".equals(a.getStatus()))
            .count();
}

private double calcExecutionCoverage(List<TestTagStatisticsData> rawData) {

    int total = rawData.stream()
            .mapToInt(TestTagStatisticsData::getTotalTests)
            .sum();

    int ranThisPeriod = rawData.stream()
            .filter(r -> r.getPeriodIdx() == 0)
            .mapToInt(TestTagStatisticsData::getTestsWithRunInPeriod)
            .sum();

    return total == 0 ? 0 : (ranThisPeriod * 100.0) / total;
}


private double calcPassRate(List<TestTagStatisticsData> rawData) {

    int ran = rawData.stream()
            .filter(r -> r.getPeriodIdx() == 0)
            .mapToInt(TestTagStatisticsData::getTestsWithRunInPeriod)
            .sum();

    int passed = rawData.stream()
            .filter(r -> r.getPeriodIdx() == 0)
            .mapToInt(TestTagStatisticsData::getPassedTests)
            .sum();

    return ran == 0 ? 0 : (passed * 100.0) / ran;
}

private double calcActivityTrend(List<TestTagStatisticsData> rawData) {

    int p0 = sumRunsForPeriod(rawData, 0);
    int p1 = sumRunsForPeriod(rawData, 1);

    if (p0 > p1) return 1;   // UP
    if (p0 < p1) return -1;  // DOWN
    return 0;                // FLAT
}

private int sumRunsForPeriod(List<TestTagStatisticsData> data, int idx) {
    return data.stream()
            .filter(r -> r.getPeriodIdx() == idx)
            .mapToInt(TestTagStatisticsData::getTestsWithRunInPeriod)
            .sum();
}

private HeatmapMetricsResponse wrapMetrics(HeatmapMetricsDTO dto) {

    List<HeatmapMetricItemDTO> items = new ArrayList<>();

    for (Field f : dto.getClass().getDeclaredFields()) {
        f.setAccessible(true);

        HeatmapMetricsConfigField config = f.getAnnotation(HeatmapMetricsConfigField.class);
        if (config == null) continue;

        try {
            double value = ((Number) f.get(dto)).doubleValue();

            items.add(new HeatmapMetricItemDTO(
                    config.label(),
                    config.icon(),
                    String.valueOf(value)
            ));

        } catch (Exception ignored) {}
    }

    return new HeatmapMetricsResponse(items);
}



```
