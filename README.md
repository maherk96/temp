```java

public String getPeriodAnalysis(LocalDateTime userStartDate) {

    if (latestRunTime == null) {
        return "This test has never run.";
    }

    String bucketStart = periodStart.format(DateTimeFormatter.ofPattern("dd MMM yyyy"));
    String selectedStart = userStartDate.format(DateTimeFormatter.ofPattern("dd MMM yyyy"));

    return switch (periodIdx) {
        case 0 -> String.format(
                "This test ran in your selected period (%s).",
                selectedStart
        );
        case 1 -> String.format(
                "This test ran a week before your selected period (on %s). Your selected period starts on %s.",
                bucketStart, selectedStart
        );
        case 2 -> String.format(
                "This test ran two weeks before your selected period (on %s). Your selected period starts on %s.",
                bucketStart, selectedStart
        );
        case 3 -> String.format(
                "This test ran three weeks before your selected period (on %s). Your selected period starts on %s.",
                bucketStart, selectedStart
        );
        default -> String.format(
                "This test ran %d weeks before your selected period (on %s). Your selected period starts on %s.",
                periodIdx, bucketStart, selectedStart
        );
    };
}
```
