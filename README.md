```java

@Override
public WidgetConfig mergeWith(WidgetConfig defaults) {
    var d = (MostFailedTestCaseConfig) defaults;
    var merged = new MostFailedTestCaseConfig();

    // appName: prefer current if non-blank, otherwise default
    merged.setAppName(this.appName != null && !this.appName.isBlank()
            ? this.appName
            : d.appName);

    // numberOfDays: prefer current if > 0, else default's,
    // and if still invalid use a hardcoded safe default (e.g. 14)
    int mergedDays = (this.numberOfDays > 0)
            ? this.numberOfDays
            : d.numberOfDays;

    if (mergedDays <= 0) {
        mergedDays = 14; // fallback to reasonable default
    }
    merged.setNumberOfDays(mergedDays);

    // includeRegression: keep current flag (or change if you prefer default)
    merged.setIncludeRegression(this.includeRegression);

    return merged;
}

public void setNumberOfDays(int numberOfDays) {
    if (numberOfDays <= 0) {
        throw new IllegalArgumentException("numberOfDays must be positive");
    }
    this.numberOfDays = numberOfDays;
}
```
