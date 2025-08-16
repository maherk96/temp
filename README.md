```java
private void executeClosedModel() throws InterruptedException {
    int users = loadProfile.getUsers();
    long rampUp = getDurationMs(loadProfile.getRampUp());
    long holdFor = getDurationMs(loadProfile.getHoldFor());
    long warmup = loadProfile.getWarmup() != null ? getDurationMs(loadProfile.getWarmup()) : 0;
    int iterations = loadProfile.getIterations();

    System.out.printf(
        "Executing closed model with users=%d, rampUp=%dms, holdFor=%dms, warmup=%dms, iterations=%d%n",
        users, rampUp, holdFor, warmup, iterations
    );

    ExecutorService executor = Executors.newFixedThreadPool(users);
    runningExecutors.put(executionId, executor);

    AtomicBoolean cancelled = runningFlags.get(executionId);

    Thread.sleep(warmup); // optional warmup before users start

    long rampUpDelay = users > 0 ? rampUp / users : 0;

    for (int i = 0; i < users; i++) {
        int userId = i;
        executor.submit(() -> {
            try {
                Thread.sleep(rampUpDelay * userId); // stagger start
                for (int j = 0; j < iterations; j++) {
                    if (cancelled.get()) {
                        break;
                    }
                    executeRequest(userId);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
    }

    // 🔑 mark executor as "done submitting"
    executor.shutdown();

    // 🔑 wait up to holdFor
    boolean finished = executor.awaitTermination(holdFor, TimeUnit.MILLISECONDS);

    if (!finished) {
        System.out.println("HoldFor expired, forcing shutdown...");
        cancelled.set(true);
        executor.shutdownNow();
    } else {
        System.out.println("All tasks completed before holdFor expired.");
    }

    runningExecutors.remove(executionId);
}
```
