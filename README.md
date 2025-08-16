```java
private void executeClosedModel() throws InterruptedException {
    var users = loadProfile.getUsers();
    var rampUp = getDurationMs(loadProfile.getRampUp());
    var holdFor = getDurationMs(loadProfile.getHoldFor());
    var warmup = loadProfile.getWarmup() != null ? getDurationMs(loadProfile.getWarmup()) : 0;
    var iterations = loadProfile.getIterations();

    System.out.println(
        "Executing closed model with users: " + users +
        ", rampUp: " + rampUp + "ms, holdFor: " + holdFor +
        "ms, warmup: " + warmup + "ms, iterations: " + iterations
    );

    var executor = Executors.newFixedThreadPool(users);
    runningExecutors.put(executionId, executor);

    // latch counts *all* iterations across all users
    var latch = new CountDownLatch(users * iterations);

    // Optional warmup delay before test starts
    Thread.sleep(warmup);

    var rampUpDelay = rampUp / users;

    for (int i = 0; i < users; i++) {
        var userId = i;
        executor.submit(() -> {
            try {
                // stagger user start
                Thread.sleep(rampUpDelay);

                for (int j = 0; j < iterations; j++) {
                    if (runningFlags.get(executionId).get()) {
                        break; // execution cancelled
                    }
                    executeRequest(userId);
                    latch.countDown(); // decrement once per iteration
                }

            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
    }

    // Wait for either all iterations or until holdFor time expires
    latch.await(holdFor, TimeUnit.MILLISECONDS);

    if (!executor.isShutdown()) {
        executor.shutdown();
    }
    runningExecutors.remove(executionId);
}
```
