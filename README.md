```java
public class IntervalStats {
    private Instant intervalStart;
    private Instant intervalEnd;

    // per-interval only
    private int requests;
    private int successes;
    private int failures;
    private double avgLatencyMS;
    private double p95LatencyMS;

    // cumulative up to this point
    private long cumulativeRequests;
    private long cumulativeSuccesses;
    private long cumulativeFailures;
    private double cumulativeAvgLatencyMS;
    private double cumulativeP95LatencyMS;

    // getters and setters ...
}

import java.time.Instant;
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.atomic.AtomicLong;

public class MetricsCollector {

    private final String executionId;

    // --- Cumulative stats (whole run)
    private final List<Long> allLatencies = new CopyOnWriteArrayList<>();
    private final AtomicLong totalRequests = new AtomicLong();
    private final AtomicLong successCount = new AtomicLong();
    private final AtomicLong failureCount = new AtomicLong();

    // --- Interval stats (reset after each window)
    private final List<Long> intervalLatencies = Collections.synchronizedList(new ArrayList<>());
    private final AtomicLong intervalSuccess = new AtomicLong();
    private final AtomicLong intervalFailures = new AtomicLong();

    private final List<IntervalStats> intervalStats = Collections.synchronizedList(new ArrayList<>());

    private Instant startTime;
    private Instant endTime;

    public MetricsCollector(String executionId) {
        this.executionId = executionId;
    }

    public void start() {
        this.startTime = Instant.now();
    }

    public void stop() {
        this.endTime = Instant.now();
    }

    /** Called for every request */
    public void record(long latencyMs, boolean success) {
        // global (cumulative)
        allLatencies.add(latencyMs);
        totalRequests.incrementAndGet();
        if (success) successCount.incrementAndGet();
        else failureCount.incrementAndGet();

        // interval (resettable)
        intervalLatencies.add(latencyMs);
        if (success) intervalSuccess.incrementAndGet();
        else intervalFailures.incrementAndGet();
    }

    /** Called every N seconds by scheduler */
public void recordInterval(Instant start, Instant end) {
    List<Long> snapshot;
    synchronized (intervalLatencies) {
        if (intervalLatencies.isEmpty() && intervalSuccess.get() == 0 && intervalFailures.get() == 0) {
            return;
        }
        snapshot = new ArrayList<>(intervalLatencies);
        intervalLatencies.clear();
    }

    IntervalStats stats = new IntervalStats();
    stats.setIntervalStart(start);
    stats.setIntervalEnd(end);

    // per-interval
    stats.setRequests(snapshot.size());
    stats.setSuccesses((int) intervalSuccess.getAndSet(0));
    stats.setFailures((int) intervalFailures.getAndSet(0));
    stats.setAvgLatencyMS(calculateAverage(snapshot));
    stats.setP95LatencyMS(calculatePercentile(snapshot, 95));

    // cumulative so far
    stats.setCumulativeRequests(totalRequests.get());
    stats.setCumulativeSuccesses(successCount.get());
    stats.setCumulativeFailures(failureCount.get());
    stats.setCumulativeAvgLatencyMS(calculateAverage(allLatencies));
    stats.setCumulativeP95LatencyMS(calculatePercentile(allLatencies, 95));

    synchronized (intervalStats) {
        intervalStats.add(stats);
    }
}

    /** Final report after run */
    public LoadTestReport buildReport(Load load) {
        LoadTestReport report = new LoadTestReport();
        report.setExecutionId(executionId);
        report.setModel(load.getModel().toString());
        report.setUsers(load.getUsers());
        report.setIterations(load.getIterations());
        report.setArrivalRatePerSec(load.getArrivalRatePerSec());
        report.setMaxConcurrent(load.getMaxConcurrent());
        report.setRampUpMs(parseDuration(load.getRampUp()));
        report.setHoldForMs(parseDuration(load.getHoldFor()));
        report.setWarmupMs(parseDuration(load.getWarmup()));
        report.setDurationMs(parseDuration(load.getDuration()));
        report.setStartTime(startTime);
        report.setEndTime(endTime);

        // cumulative stats
        report.setTotalRequests(totalRequests.get());
        report.setSuccessCount(successCount.get());
        report.setFailureCount(failureCount.get());
        report.setAvgLatencyMS(calculateAverage(allLatencies));
        report.setP95LatencyMS(calculatePercentile(allLatencies, 95));
        report.setP99LatencyMS(calculatePercentile(allLatencies, 99));
        report.setFailureRatePercent(
            totalRequests.get() == 0 ? 0.0 : (failureCount.get() * 100.0) / totalRequests.get()
        );

        // per-interval snapshots
        synchronized (intervalStats) {
            report.setPerIntervalStats(new ArrayList<>(intervalStats));
        }

        return report;
    }

    // ---- Helpers ----
    private double calculateAverage(List<Long> values) {
        if (values.isEmpty()) return 0;
        return values.stream().mapToLong(Long::longValue).average().orElse(0);
    }

    private double calculatePercentile(List<Long> values, int percentile) {
        if (values.isEmpty()) return 0;
        List<Long> sorted = new ArrayList<>(values);
        Collections.sort(sorted);
        int index = (int) Math.ceil(percentile / 100.0 * sorted.size()) - 1;
        return sorted.get(Math.max(index, 0));
    }

    private long parseDuration(String value) {
        if (value == null) return 0;
        if (value.endsWith("ms")) return Long.parseLong(value.replace("ms", ""));
        if (value.endsWith("s")) return Long.parseLong(value.replace("s", "")) * 1000;
        if (value.endsWith("m")) return Long.parseLong(value.replace("m", "")) * 60 * 1000;
        return Long.parseLong(value); // assume ms
    }
}
```
