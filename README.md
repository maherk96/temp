```java
private List<HeatmapAnalysisDTO> buildHeatmapResponse(
        List<TestTagStatisticsData> data,
        LocalDateTime userStartDate
) {
    DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd MMM yyyy");

    // 1️⃣ For each (className, tagName), keep the row with the MOST RECENT latestRunTime
    List<TestTagStatisticsData> mostRecentPerCell =
            data.stream()
                .collect(Collectors.groupingBy(
                        row -> row.getClassName() + "|" + row.getTagName(),
                        Collectors.maxBy(Comparator.comparing(TestTagStatisticsData::getLatestRunTime))
                ))
                .values()
                .stream()
                .map(Optional::get)
                .toList();

    // 2️⃣ Map to DTOs using latestRunTime vs userStartDate
    return mostRecentPerCell.stream()
        .map(row -> {
            LocalDateTime lastRun = row.getLatestRunTime();

            HeatmapStatus status = determineStatus(lastRun, userStartDate);
            String lastRunStr = lastRun != null
                    ? lastRun.toLocalDate().format(fmt)
                    : "N/A";

            String periodAnalysis = buildPeriodAnalysis(lastRun, userStartDate);

            return new HeatmapAnalysisDTO(
                    row.getClassName(),
                    row.getTagName(),
                    row.getPassedTests(),
                    row.getPassRate(),
                    row.getTotalTests(),
                    status,
                    lastRunStr,
                    periodAnalysis
            );
        })
        .toList();
}

private HeatmapStatus determineStatus(LocalDateTime lastRun, LocalDateTime userStartDate) {
    if (lastRun == null) {
        return HeatmapStatus.STALE;
    }
    // ACTIVE if last run is on or after the user's selected start date
    return lastRun.isBefore(userStartDate)
            ? HeatmapStatus.STALE
            : HeatmapStatus.ACTIVE;
}

private String buildPeriodAnalysis(LocalDateTime lastRun, LocalDateTime userStartDate) {
    if (lastRun == null) {
        return "This test has never run.";
    }

    DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd MMM yyyy");

    LocalDate runDate = lastRun.toLocalDate();
    LocalDate selectedStart = userStartDate.toLocalDate();

    String runStr = runDate.format(fmt);
    String selectedStr = selectedStart.format(fmt);

    // In or after selected period
    if (!runDate.isBefore(selectedStart)) {
        return String.format(
                "This test ran in your selected period (%s).",
                selectedStr
        );
    }

    long weeks = ChronoUnit.WEEKS.between(runDate, selectedStart);

    return switch ((int) weeks) {
        case 1 -> String.format(
                "This test ran a week before your selected period (on %s). Your selected period starts on %s.",
                runStr, selectedStr
        );
        case 2 -> String.format(
                "This test ran two weeks before your selected period (on %s). Your selected period starts on %s.",
                runStr, selectedStr
        );
        case 3 -> String.format(
                "This test ran three weeks before your selected period (on %s). Your selected period starts on %s.",
                runStr, selectedStr
        );
        default -> String.format(
                "This test ran %d weeks before your selected period (on %s). Your selected period starts on %s.",
                weeks, runStr, selectedStr
        );
    };
}



```
