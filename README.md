```java

private HeatmapMetricsResponse wrapMetrics(HeatmapMetricsDTO dto) {

    List<HeatmapMetricItemDTO> items = new ArrayList<>();

    for (Field field : dto.getClass().getDeclaredFields()) {
        field.setAccessible(true);

        HeatmapMetricsConfigField config =
                field.getAnnotation(HeatmapMetricsConfigField.class);
        if (config == null) continue;

        try {
            Object rawValue = field.get(dto);
            double value = rawValue instanceof Number ?
                    ((Number) rawValue).doubleValue() : 0;

            String icon = determineIcon(field.getName(), value);

            items.add(new HeatmapMetricItemDTO(
                    config.label(),
                    icon,
                    String.valueOf(value)
            ));

        } catch (Exception ignored) {}
    }

    return new HeatmapMetricsResponse(items);
}

private String determineIcon(String fieldName, double value) {

    // Special logic only for activityTrend
    if ("activityTrend".equals(fieldName)) {

        if (Double.isInfinite(value)) {
            return "infinite";   // Your UI can display ∞
        }

        if (value > 0) {
            return "arrow_up";  // trending up
        }

        if (value < 0) {
            return "arrow_down"; // trending down
        }

        return "trending_flat"; // 0 => unchanged
    }

    // Default icons for other metrics (optional upgrade later)
    switch (fieldName) {
        case "freshnessScore": return "refresh";
        case "staleCount": return "warning";
        case "executionCoverage": return "coverage";
        case "passRate": return "check";
    }

    return "info";
}

```
