```java
private LoadTestReport executeClosedModel() throws InterruptedException {
    int users = loadProfile.getUsers();
    long rampUp = toMillis(loadProfile.getRampUp());
    long holdFor = toMillis(loadProfile.getHoldFor());
    long warmup = toMillis(loadProfile.getWarmup());
    int iterations = loadProfile.getIterations();

    logger.info(
        "Executing CLOSED model: users={}, rampUp={}ms, holdFor={}ms, warmup={}ms, iterations={}, sla={}",
        users, rampUp, holdFor, warmup, iterations, loadProfile.getServiceLevelAgreement()
    );

    ScheduledExecutorService metricsScheduler = startMetricsRecorder();
    ExecutorService executor = Executors.newFixedThreadPool(users);
    runningExecutors.put(executionId, executor);
    AtomicBoolean cancelled = runningFlags.get(executionId);

    try {
        // Warmup delay
        Thread.sleep(warmup);

        // Spread user start-up across ramp-up period
        long rampUpDelay = users > 0 ? rampUp / users : 0;

        for (int i = 0; i < users; i++) {
            if (shouldStop()) {
                logger.warn("Execution {} cancelled before starting user {}", executionId, i);
                break;
            }
            final int userId = i;
            executor.submit(() -> runUserClosedLoad(userId, iterations, rampUpDelay, cancelled));
        }

        // Main loop: enforce hold time, SLA, and early completion
        long deadline = System.currentTimeMillis() + holdFor;
        while (System.currentTimeMillis() < deadline) {
            // Exit early if all tasks done
            if (executor.isTerminated()) {
                logger.info("Execution {} completed successfully", executionId);
                break;
            }

            // Exit early if SLA breach or cancelled
            if (shouldStop()) {
                logger.warn("Execution {} stopping early due to SLA breach or cancellation", executionId);
                cancelled.set(true);
                executor.shutdownNow();
                break;
            }

            // Wait a bit before checking again
            executor.awaitTermination(1, TimeUnit.SECONDS);
        }

        // If still running after holdFor, stop forcefully
        if (!executor.isTerminated()) {
            logger.warn("Execution {} timed out after {}ms", executionId, holdFor);
            cancelled.set(true);
            executor.shutdownNow();
        }

    } finally {
        shutdownExecutor(executor);
        shutdownExecutor(metricsScheduler);
        runningExecutors.remove(executionId);
    }

    return generateReport();
}
```
