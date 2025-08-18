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
    
    // Track active tasks
    AtomicInteger activeTasks = new AtomicInteger(0);
    
    try {
        Thread.sleep(warmup);
        long rampUpDelay = users > 0 ? rampUp / users : 0;
        
        for (int i = 0; i < users; i++) {
            if (isCancelled()) {
                logger.warn("Execution {} cancelled before starting user {}", executionId, i);
                break;
            }
            
            // Check SLA breach BEFORE starting tasks - false means breach
            if (!metricsCollector.getIsRunning().get()) {
                logger.warn("Execution {} stopping early due to SLA breach", executionId);
                break;
            }
            
            final int userId = i;
            activeTasks.incrementAndGet();
            executor.submit(() -> {
                try {
                    runUserClosedLoad(userId, iterations, rampUpDelay, cancelled);
                } finally {
                    activeTasks.decrementAndGet();
                }
            });
        }
        
        long startTime = System.currentTimeMillis();
        long endTime = startTime + holdFor;
        
        while (System.currentTimeMillis() < endTime) {
            // Check if SLA is breached - false means breach, terminate immediately
            if (!metricsCollector.getIsRunning().get()) {
                logger.warn("Execution {} stopping early due to SLA breach", executionId);
                executor.shutdownNow();
                break;
            }
            
            // Check if all tasks are completed - can terminate early
            if (activeTasks.get() == 0) {
                logger.info("Execution {} completed as all tasks finished", executionId);
                break;
            }
            
            // Brief sleep to avoid busy waiting
            Thread.sleep(100);
        }
        
        // If we're here and still have active tasks, holdFor time has passed
        if (activeTasks.get() > 0) {
            logger.warn("Execution {} timed out after {}ms", executionId, holdFor);
            cancelled.set(true);
            executor.shutdownNow();
        }
        
        // Wait a reasonable time for executor to terminate
        if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
            logger.warn("Execution {} did not terminate gracefully", executionId);
        }
        
    } finally {
        shutdownExecutor(executor);
        shutdownExecutor(metricsScheduler);
        runningExecutors.remove(executionId);
    }
    
    return generateReport();
}
```
