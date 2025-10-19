```java
try (var cursor = env.getServiceFactory()
        .getQueueCursor()
        .decode(ExecutionReportListener.class)
        .forQueue("order-processor")) {

    var reports = waitForAllExecutionReports(cursor);

    System.out.println("Received " + reports.size() + " execution reports:");
    reports.forEach(r -> System.out.println(" -> " + r.getClOrdID()));
}
```
