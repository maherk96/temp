```java

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Duration;
import java.time.Instant;
import java.util.Map;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicBoolean;

/**
 * The {@code ClientLoadExecutor} is responsible for executing load tests against REST endpoints
 * based on a given {@link Load} profile. It supports both {@code CLOSED} and {@code OPEN} load models:
 *
 * <ul>
 *     <li><b>Closed Model:</b> Fixed number of users executing a fixed number of iterations with ramp-up and hold times.</li>
 *     <li><b>Open Model:</b> Arrival rate–driven execution, where requests arrive at a specified rate regardless of completion time.</li>
 * </ul>
 *
 * The executor manages concurrent executions, tracks running state using flags, collects metrics,
 * and generates a {@link LoadTestReport} summarizing the execution.
 *
 * Thread-safe execution is ensured using {@link ConcurrentHashMap} and {@link AtomicBoolean}.
 */
public class ClientLoadExecutor {

    private static final Logger logger = LoggerFactory.getLogger(ClientLoadExecutor.class);

    /** Timeout (in seconds) used when shutting down executors gracefully. */
    private static final int EXECUTOR_SHUTDOWN_TIMEOUT_SEC = 30;

    private final RestBenchHttpClient clientExecutor;
    private final Load loadProfile;
    private final String executionId;
    private final Map<String, ExecutorService> runningExecutors = new ConcurrentHashMap<>();
    private final Map<String, AtomicBoolean> runningFlags = new ConcurrentHashMap<>();
    private final RestExecSpec request;
    private final MetricsCollector metricsCollector;

    /**
     * Constructs a new {@code ClientLoadExecutor}.
     *
     * @param clientExecutor the HTTP client executor used to send requests
     * @param loadProfile    the load test profile configuration
     * @param executionId    unique identifier for this execution
     * @param request        the REST execution specification containing requests and flows
     */
    public ClientLoadExecutor(RestBenchHttpClient clientExecutor, Load loadProfile, String executionId, RestExecSpec request) {
        this.clientExecutor = clientExecutor;
        this.loadProfile = loadProfile;
        this.executionId = executionId;
        this.request = request;
        runningFlags.computeIfAbsent(executionId, k -> new AtomicBoolean(false));
        this.metricsCollector = new MetricsCollector(request.getRequestId());
    }

    /**
     * Executes the load test based on the configured load model.
     *
     * @return a {@link LoadTestReport} summarizing the execution results
     * @throws InterruptedException if the execution is interrupted
     */
    public LoadTestReport executeRequest() throws InterruptedException {
        switch (loadProfile.getModel()) {
            case OPEN:
                return executeOpenModel();
            case CLOSED:
                return executeClosedModel();
            default:
                throw new IllegalArgumentException("Unsupported load model: " + loadProfile.getModel());
        }
    }

    /**
     * Cancels a running execution by shutting down its associated executor and
     * updating its cancellation flag.
     *
     * @param executionId the identifier of the execution to cancel
     */
    public void cancelExecution(String executionId) {
        try {
            var flag = runningFlags.get(executionId);
            if (flag != null) {
                flag.set(true);
                var executor = runningExecutors.get(executionId);
                if (executor != null) {
                    executor.shutdownNow();
                    runningExecutors.remove(executionId);
                }
            }
        } catch (Exception e) {
            throw new RuntimeException("Failed to cancel execution for ID: " + executionId, e);
        }
    }

    /**
     * Executes a <b>closed workload model</b>:
     * <ul>
     *     <li>A fixed number of users</li>
     *     <li>Each user runs for a fixed number of iterations</li>
     *     <li>Optional ramp-up delay and warmup period</li>
     *     <li>Test duration enforced via {@code holdFor}</li>
     * </ul>
     *
     * @return a {@link LoadTestReport} with the execution summary
     * @throws InterruptedException if interrupted during execution
     */
    private LoadTestReport executeClosedModel() throws InterruptedException {
        int users = loadProfile.getUsers();
        long rampUp = toMillis(loadProfile.getRampUp());
        long holdFor = toMillis(loadProfile.getHoldFor());
        long warmup = loadProfile.getWarmup() != null ? toMillis(loadProfile.getWarmup()) : 0;
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
                final int userId = i;
                executor.submit(() -> runUserClosedLoad(userId, iterations, rampUpDelay, cancelled));
            }

            boolean finished = executor.awaitTermination(holdFor, TimeUnit.MILLISECONDS);

            if (!finished) {
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

    /**
     * Executes an <b>open workload model</b>:
     * <ul>
     *     <li>Requests arrive at a given rate per second</li>
     *     <li>Maximum concurrent requests are limited by a semaphore</li>
     *     <li>Test duration is fixed</li>
     * </ul>
     *
     * @return a {@link LoadTestReport} with the execution summary
     * @throws InterruptedException if interrupted during execution
     */
    private LoadTestReport executeOpenModel() throws InterruptedException {
        int arrivalRatePerSec = loadProfile.getArrivalRatePerSec();
        int maxConcurrent = loadProfile.getMaxConcurrent();
        long durationMs = toMillis(loadProfile.getDuration());
        long warmupMs = toMillis(loadProfile.getWarmup());

        ExecutorService executor = Executors.newCachedThreadPool();
        runningExecutors.put(executionId, executor);
        Semaphore semaphore = new Semaphore(maxConcurrent);

        long endTime = System.currentTimeMillis() + durationMs;

        logger.info("Executing OPEN model: arrivalRate={} req/s, maxConcurrent={}, duration={}ms, warmup={}ms",
                arrivalRatePerSec, maxConcurrent, durationMs, warmupMs);

        ScheduledExecutorService metricsScheduler = startMetricsRecorder();
        Thread.sleep(warmupMs);

        try {
            while (System.currentTimeMillis() < endTime && !isCancelled()) {
                for (int i = 0; i < arrivalRatePerSec; i++) {
                    if (isCancelled() || System.currentTimeMillis() >= endTime) {
                        break;
                    }

                    semaphore.acquire();
                    final int userId = i;

                    executor.submit(() -> {
                        try {
                            if (isCancelled()) return;
                            executeRequest(userId);
                            sleepWithCancelCheck(getThinkTimeMs());
                        } catch (InterruptedException e) {
                            Thread.currentThread().interrupt();
                        } finally {
                            semaphore.release();
                        }
                    });
                }
                Thread.sleep(1000);
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

    /**
     * Runs the closed workload model logic for a single user.
     *
     * @param userId       the user identifier
     * @param iterations   number of iterations to execute
     * @param rampUpDelay  delay per user during ramp-up
     * @param cancelled    cancellation flag
     */
    private void runUserClosedLoad(int userId, int iterations, long rampUpDelay, AtomicBoolean cancelled) {
        try {
            Thread.sleep(rampUpDelay * userId);
            for (int j = 0; j < iterations && !cancelled.get(); j++) {
                executeRequest(userId);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    /**
     * Executes all requests for a single user in one iteration.
     *
     * @param userId the user identifier
     */
    private void executeRequest(int userId) {
        if (isCancelled()) return;

        request.getRequests().forEach(req -> {
            var response = clientExecutor.execute(req);
            metricsCollector.record(
                    response.getResponseTimeMs(),
                    response.getStatusCode() >= 200 && response.getStatusCode() < 300
            );
            logger.debug("Response [userId={}]: status={}, time={}ms",
                    userId, response.getStatusCode(), response.getResponseTimeMs());
        });
    }

    /**
     * Converts a duration string into milliseconds.
     * <p>
     * Supported formats:
     * <ul>
     *     <li>"10ms" → 10</li>
     *     <li>"10s" → 10000</li>
     *     <li>"5m" → 300000</li>
     *     <li>"2h" → 7200000</li>
     *     <li>ISO-8601 format (e.g., "PT10S")</li>
     * </ul>
     *
     * @param duration the duration string
     * @return duration in milliseconds
     */
    private long toMillis(String duration) {
        if (duration == null || duration.isBlank()) return 0L;
        if (duration.matches("\\d+(ms|s|m|h)")) {
            if (duration.endsWith("ms")) return Long.parseLong(duration.replace("ms", ""));
            if (duration.endsWith("s")) return Long.parseLong(duration.replace("s", "")) * 1000;
            if (duration.endsWith("m")) return Long.parseLong(duration.replace("m", "")) * 60 * 1000;
            if (duration.endsWith("h")) return Long.parseLong(duration.replace("h", "")) * 60 * 60 * 1000;
        }
        return Duration.parse(duration).toMillis();
    }

    /**
     * Starts a recurring metrics recorder that logs intervals every 5 seconds.
     *
     * @return the {@link ScheduledExecutorService} managing the recording task
     */
    private ScheduledExecutorService startMetricsRecorder() {
        ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
        scheduler.scheduleAtFixedRate(() -> {
            Instant now = Instant.now();
            metricsCollector.recordInterval(now.minusSeconds(5), now);
        }, 5, 5, TimeUnit.SECONDS);
        return scheduler;
    }

    /**
     * Generates a random think time (pause) between requests.
     *
     * @return a random think time in milliseconds
     */
    private int getThinkTimeMs() {
        return ThreadLocalRandom.current().nextInt(
                request.getFlow().getThinkTimeMs().getMin(),
                request.getFlow().getThinkTimeMs().getMax()
        );
    }

    /**
     * Sleeps for the specified duration but periodically checks whether the execution has been cancelled.
     *
     * @param millis the maximum sleep duration in milliseconds
     * @throws InterruptedException if the thread is interrupted
     */
    private void sleepWithCancelCheck(long millis) throws InterruptedException {
        long end = System.currentTimeMillis() + millis;
        while (System.currentTimeMillis() < end && !isCancelled()) {
            Thread.sleep(Math.min(100, end - System.currentTimeMillis()));
        }
    }

    /**
     * Shuts down the given executor gracefully, falling back to {@code shutdownNow()}
     * if it does not terminate in time.
     *
     * @param executor the executor to shut down
     */
    private void shutdownExecutor(ExecutorService executor) {
        if (executor == null) return;
        executor.shutdown();
        try {
            if (!executor.awaitTermination(EXECUTOR_SHUTDOWN_TIMEOUT_SEC, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            executor.shutdownNow();
        }
    }

    /**
     * Checks whether the current execution has been cancelled.
     *
     * @return true if cancelled, false otherwise
     */
    private boolean isCancelled() {
        return runningFlags.getOrDefault(executionId, new AtomicBoolean(false)).get();
    }
}
```
