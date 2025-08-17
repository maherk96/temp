```java
import java.time.Instant;
import java.util.List;
import java.util.Map;

public class LoadTestReport {
    private String executionId;
    private LoadModel model;
    private int users;
    private int iterations;
    private int arrivalRatePerSec;
    private int maxConcurrent;

    private long rampUpMs;
    private long holdForMs;
    private long warmupMs;
    private long durationMs;

    private Instant startTime;
    private Instant endTime;

    private long totalRequests;
    private long successCount;
    private long failureCount;

    private double avgLatencyMs;
    private double p95LatencyMs;
    private double p99LatencyMs;

    private double failureRatePercent;

    private List<IntervalStats> perIntervalStats;
    private Map<String, Object> extra;

    // --- getters & setters ---
    // (Generate via Lombok @Data if you like)
}

import java.time.Instant;

public class IntervalStats {
    private Instant intervalStart;
    private Instant intervalEnd;
    private long requests;
    private long successes;
    private long failures;
    private double avgLatencyMs;
    private double p95LatencyMs;

    // getters & setters
}

import java.time.Instant;
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.atomic.AtomicLong;

public class MetricsCollector {
    private final String executionId;
    private final List<Long> latencies = new CopyOnWriteArrayList<>();
    private final AtomicLong totalRequests = new AtomicLong();
    private final AtomicLong successCount = new AtomicLong();
    private final AtomicLong failureCount = new AtomicLong();

    private final List<IntervalStats> intervalStats = new ArrayList<>();
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

    public void record(long latencyMs, boolean success) {
        latencies.add(latencyMs);
        totalRequests.incrementAndGet();
        if (success) successCount.incrementAndGet();
        else failureCount.incrementAndGet();
    }

public void recordInterval(Instant start, Instant end) {
    long total = successCount.get() + failureCount.get();
    if (total == 0) return;

    IntervalStats stats = new IntervalStats();
    stats.setIntervalStart(start);
    stats.setIntervalEnd(end);
    stats.setRequests(total);
    stats.setSuccesses(successCount.get());
    stats.setFailures(failureCount.get());
    stats.setAvgLatencyMs(calculateAverage(latencies));
    stats.setP95LatencyMs(calculatePercentile(latencies, 95));

    intervalStats.add(stats);
}


    public LoadTestReport buildReport(Load load) {
        LoadTestReport report = new LoadTestReport();
        report.setExecutionId(executionId);
        report.setModel(load.getModel());
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

        report.setTotalRequests(totalRequests.get());
        report.setSuccessCount(successCount.get());
        report.setFailureCount(failureCount.get());

        report.setAvgLatencyMs(calculateAverage(latencies));
        report.setP95LatencyMs(calculatePercentile(latencies, 95));
        report.setP99LatencyMs(calculatePercentile(latencies, 99));

        report.setFailureRatePercent(
            totalRequests.get() == 0 ? 0.0 :
            (failureCount.get() * 100.0) / totalRequests.get()
        );

        report.setPerIntervalStats(intervalStats);

        return report;
    }

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
        return Long.parseLong(value); // default ms
    }
}


```
