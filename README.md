```java
public String getPeriodData(LocalDateTime userStartDate) {
    if (latestRunTime == null) {
        return "This test has never run";
    }

    String datePrefix = latestRunTime.toLocalDate()
            .format(DateTimeFormatter.ofPattern("dd MMM yyyy"));

    // if last run is on or after the user's selected start date → period 0
    if (!latestRunTime.isBefore(userStartDate)) {
        return String.format("This test ran in your selected period (%s)", 
                userStartDate.toLocalDate().format(DateTimeFormatter.ofPattern("dd MMM yyyy")));
    }

    // calculate weeks difference
    long weeks = ChronoUnit.WEEKS.between(latestRunTime.toLocalDate(), userStartDate.toLocalDate());

    if (weeks <= 0) {
        return String.format("This test ran in your selected period (%s)", datePrefix);
    } else if (weeks == 1) {
        return String.format("This test ran a week before (%s)", datePrefix);
    } else if (weeks == 2) {
        return String.format("This test ran two weeks before (%s)", datePrefix);
    } else if (weeks == 3) {
        return String.format("This test ran three weeks before (%s)", datePrefix);
    }

    return String.format("This test ran %d weeks before (%s)", weeks, datePrefix);
}
```
