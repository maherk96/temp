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

    // pool = one thread per "virtual user"
    ExecutorService executor = Executors.newFixedThreadPool(users);
    runningExecutors.put(executionId, executor);

    // flag for stopping when holdFor expires
    AtomicBoolean cancelled = runningFlags.get(executionId);

    // Optional warmup delay before starting
    Thread.sleep(warmup);

    long rampUpDelay = rampUp / users;

    for (int i = 0; i < users; i++) {
        int userId = i;
        executor.submit(() -> {
            try {
                Thread.sleep(rampUpDelay * userId); // stagger users
                for (int j = 0; j < iterations; j++) {
                    if (cancelled.get()) {
                        break; // execution cancelled
                    }
                    executeRequest(userId);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt(); // thread was killed
            }
        });
    }

    // Wait for holdFor period, then cancel remaining tasks
    if (!executor.awaitTermination(holdFor, TimeUnit.MILLISECONDS)) {
        System.out.println("HoldFor expired, forcing shutdown...");
        cancelled.set(true);
        executor.shutdownNow(); // kill tasks immediately
    }

    runningExecutors.remove(executionId);
}
```
