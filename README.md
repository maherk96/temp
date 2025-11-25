```java
public String getPeriodData() {
    var datePrefix = periodStart != null
            ? periodStart.toLocalDate().toString()
            : "N/A";

    return switch (periodIdx) {
        case 0 -> String.format("This test ran in your current period (%s)", datePrefix);
        case 1 -> String.format("This test ran a week before (%s)", datePrefix);
        case 2 -> String.format("This test ran two weeks before (%s)", datePrefix);
        case 3 -> String.format("This test ran three weeks before (%s)", datePrefix);
        default -> {
            String weekWord = periodIdx == 1 ? "week" : "weeks";
            yield String.format("This test ran %d %s before (%s)",
                    periodIdx, weekWord, datePrefix);
        }
    };
}

```
