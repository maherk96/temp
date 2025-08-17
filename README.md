```java
private void executeOpenModel() throws InterruptedException {
    int arrivalRatePerSec = profile.getArrivalRatePerSec(); // requests per second
    int maxConcurrent = profile.getMaxConcurrent();         // max concurrent requests
    int durationMs = parseDurationToMs(profile.getDuration());
    int warmupMs = parseDurationToMs(profile.getWarmup());

    ExecutorService executor = Executors.newCachedThreadPool();
    runningExecutors.put(executionId, executor);

    Semaphore semaphore = new Semaphore(maxConcurrent);
    long endTime = System.currentTimeMillis() + durationMs;

    System.out.println("Executing open model with arrivalRate=" + arrivalRatePerSec
            + " req/s, maxConcurrent=" + maxConcurrent
            + ", duration=" + durationMs + "ms, warmup=" + warmupMs + "ms");

    // Warmup phase
    Thread.sleep(warmupMs);

    try {
        while (System.currentTimeMillis() < endTime && !cancellationFlags.get(executionId).get()) {
            // Submit requests at the defined arrival rate
            for (int i = 0; i < arrivalRatePerSec; i++) {
                if (cancellationFlags.get(executionId).get() || System.currentTimeMillis() >= endTime) {
                    break;
                }

                semaphore.acquire(); // ensure we don’t exceed maxConcurrent

                int userId = i; // simple user id
                executor.submit(() -> {
                    try {
                        if (cancellationFlags.get(executionId).get()) {
                            return;
                        }
                        executeRequests(userId);       // your actual request logic
                        Thread.sleep(getThinkTimeMs()); // think time between requests
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    } finally {
                        semaphore.release(); // free a permit
                    }
                });
            }

            Thread.sleep(1000); // pace the loop to 1-second chunks
        }
    } finally {
        // Shutdown executor gracefully
        executor.shutdown();
        if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
            executor.shutdownNow();
        }
        runningExecutors.remove(executionId);
        System.out.println("Execution ended for open model.");
    }
}
```
