```java
private LoadTestReport executeClosedModel() throws InterruptedException {
    int users = loadProfile.getUsers();
    long rampUp = toMillis(loadProfile.getRampUp());
    long holdFor = toMillis(loadProfile.getHoldFor());
    long warmup = loadProfile.getWarmup() != null ? toMillis(loadProfile.getWarmup()) : 0;
    int iterations = loadProfile.getIterations();

    logger.info("Executing CLOSED model: users={}, rampUp={}ms, holdFor={}ms, warmup={}ms, iterations={}, sla={}",
            users, rampUp, holdFor, warmup, iterations, loadProfile.getServiceLevelAgreement());

    ScheduledExecutorService metricsScheduler = startMetricsRecorder();
    ExecutorService executor = Executors.newFixedThreadPool(users);
    runningExecutors.put(executionId, executor);
    AtomicBoolean cancelled = runningFlags.get(executionId);

    try {
        Thread.sleep(warmup);
        long rampUpDelay = users > 0 ? rampUp / users : 0;

        for (int i = 0; i < users; i++) {
            // 🚨 EARLY EXIT if SLA failed
            if (isCancelled() || !metricsCollector.getIsRunning().get()) {
                logger.warn("Execution {} cancelled before starting user {}", executionId, i);
                break;
            }

            final int userId = i;
            executor.submit(() -> runUserClosedLoad(userId, iterations, rampUpDelay, cancelled));
        }

        // 🚨 Instead of blindly waiting, check SLA flag
        while (!executor.isTerminated()) {
            if (!metricsCollector.getIsRunning().get()) {
                logger.warn("Execution {} stopping early due to SLA breach", executionId);
                executor.shutdownNow(); // kill active tasks
                break;
            }
            if (executor.awaitTermination(1, TimeUnit.SECONDS)) {
                break; // finished naturally
            }
        }

        if (!executor.isTerminated()) {
            logger.warn("Execution {} timed out after {}ms", executionId, holdFor);
            cancelled.set(true);
            executor.shutdownNow();
        } else {
            logger.info("Execution completed for {}", executionId);
        }

    } finally {
        shutdownExecutor(executor);
        shutdownExecutor(metricsScheduler);
        runningExecutors.remove(executionId);
    }

    LoadTestReport report = metricsCollector.buildReport(loadProfile);
    logger.info("Report for {}: {}", executionId, report);
    return report;
}

```
