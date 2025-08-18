```java
private LoadTestReport executeClosedModel() throws InterruptedException {
    int users = loadProfile.getUsers();
    long rampUp = toMillis(loadProfile.getRampUp());
    long holdFor = toMillis(loadProfile.getHoldFor());
    long warmup = toMillis(loadProfile.getWarmup());
    int iterations = loadProfile.getIterations();

    logger.info("Executing CLOSED model: users={}, rampUp={}ms, holdFor={}ms, warmup={}ms, iterations={}",
            users, rampUp, holdFor, warmup, iterations);

    ScheduledExecutorService metricsScheduler = startMetricsRecorder();
    ExecutorService executor = Executors.newFixedThreadPool(users);
    runningExecutors.put(executionId, executor);
    AtomicBoolean cancelled = runningFlags.get(executionId);

    try {
        Thread.sleep(warmup);
        long rampUpDelay = users > 0 ? rampUp / users : 0;

        for (int i = 0; i < users; i++) {
            if (shouldStop()) break;
            final int userId = i;
            executor.submit(() -> runUserClosedLoad(userId, iterations, rampUpDelay, cancelled));
        }

        // ✅ Wait only for hold time
        boolean finished = executor.awaitTermination(holdFor, TimeUnit.MILLISECONDS);
        if (!finished) {
            logger.warn("Execution {} timed out after {}ms", executionId, holdFor);
            cancelled.set(true);
            executor.shutdownNow();
        } else {
            logger.info("Execution {} completed successfully", executionId);
        }

    } finally {
        shutdownExecutor(executor);
        shutdownExecutor(metricsScheduler);
        runningExecutors.remove(executionId);
    }

    return generateReport();
}

private boolean shouldStop() {
    AtomicBoolean cancelled = runningFlags.get(executionId);
    return (cancelled != null && cancelled.get()) 
           || !metricsCollector.getIsRunning().get();
}
```
