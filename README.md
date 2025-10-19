```java
protected List<DefaultExecutionReport> waitForAllExecutionReports(QueueCursor cursor) {
    List<DefaultExecutionReport> reports = new ArrayList<>();

    // Keep decoding while new entries are available
    while (waitForCondition(cursor::nextDecodable)) {
        if (cursor.entry().methodId() == MethodIDs.EXECUTION_REPORT_LISTENER_ON_EXECUTION_REPORT) {
            // Argument index 1 = DefaultExecutionReport (since index 0 = sessionId)
            var report = (DefaultExecutionReport) cursor.entry().getArgument(1);
            reports.add(report);
        }
    }

    return reports;
}
```
