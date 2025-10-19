```java
protected List<DefaultExecutionReport> waitForAllExecutionReports(QueueCursor cursor) {
    List<DefaultExecutionReport> reports = new ArrayList<>();

    // Keep trying to decode entries until timeout or queue is empty
    while (true) {
        try {
            // This will block until nextDecodable() returns true or timeout expires
            waitForCondition(cursor::nextDecodable);

            // If we reached here, there’s a decodable entry ready
            if (cursor.entry().methodId() == MethodIDs.EXECUTION_REPORT_LISTENER_ON_EXECUTION_REPORT) {
                var report = (DefaultExecutionReport) cursor.entry().getArgument(1);
                reports.add(report);
            }

        } catch (RuntimeException timeout) {
            // "timeout exceeded" means no new entries arrived — stop collecting
            break;
        }
    }

    return reports;
}
```
