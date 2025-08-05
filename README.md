```java
// File: EventLatencyRecorder.java

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import net.openhft.chronicle.queue.ChronicleQueue;
import net.openhft.chronicle.queue.ExcerptAppender;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

/**
 * EventLatencyRecorder records latencies between protocol events (e.g. FIX, REST) and their responses.
 * <p>
 * Features:
 * <ul>
 *     <li>Auto-generates unique request IDs per event lifecycle.</li>
 *     <li>Captures latencies using Micrometer, tagged with protocol, from, to, and success state.</li>
 *     <li>Writes raw event data to Chronicle Queue for later replay/debugging.</li>
 *     <li>Supports multiple concurrent requests safely via thread-safe structures.</li>
 * </ul>
 */
public class EventLatencyRecorder {

    private final MeterRegistry registry;
    private final Map<Long, List<EventCheckpoint>> activeRequests = new ConcurrentHashMap<>();
    private final AtomicLong idGenerator = new AtomicLong(1);
    private final ExcerptAppender appender;

    /**
     * Constructs an EventLatencyRecorder.
     *
     * @param registry  the Micrometer MeterRegistry to publish metrics
     * @param queuePath the path where Chronicle Queue stores raw event data
     */
    public EventLatencyRecorder(MeterRegistry registry, String queuePath) {
        this.registry = Objects.requireNonNull(registry, "MeterRegistry cannot be null");
        ChronicleQueue queue = ChronicleQueue.singleBuilder(queuePath).build();
        this.appender = queue.acquireAppender();
    }

    /**
     * Starts tracking a new request lifecycle.
     *
     * @param protocol  the protocol (e.g. FIX, REST)
     * @param eventType the initial event type (e.g. NOS, POST)
     * @return a unique request ID for this lifecycle
     */
    public long start(String protocol, String eventType) {
        long id = idGenerator.getAndIncrement();
        long now = System.nanoTime();
        List<EventCheckpoint> events = new ArrayList<>();
        events.add(new EventCheckpoint(eventType, now));
        activeRequests.put(id, events);

        writeRawEvent(id, protocol, eventType, now, true);
        return id;
    }

    /**
     * Marks a new event stage in the lifecycle.
     * Records latencies from the previous stage and from the start stage to this stage.
     *
     * @param requestId    the unique request ID
     * @param protocol     the protocol (e.g. FIX, REST)
     * @param eventResponse the current event/stage (e.g. PENDING_NEW, NEW, FILL)
     * @param success      whether the event was successful
     * @param finalStage   whether this marks the final stage of the lifecycle (removes request from tracking)
     */
    public void mark(long requestId, String protocol, String eventResponse, boolean success, boolean finalStage) {
        List<EventCheckpoint> events = activeRequests.get(requestId);
        if (events == null) return;

        long now = System.nanoTime();
        EventCheckpoint current = new EventCheckpoint(eventResponse, now);
        EventCheckpoint previous = events.get(events.size() - 1);
        events.add(current);

        // Record latency from previous to current
        recordLatency(protocol, previous.name, current.name, now - previous.timestamp, success);

        // Record latency from start to current
        EventCheckpoint start = events.get(0);
        if (!start.name.equals(previous.name)) {
            recordLatency(protocol, start.name, current.name, now - start.timestamp, success);
        }

        writeRawEvent(requestId, protocol, eventResponse, now, success);

        if (finalStage) {
            activeRequests.remove(requestId);
        }
    }

    /**
     * Records a latency sample to Micrometer.
     */
    private void recordLatency(String protocol, String from, String to, long durationNanos, boolean success) {
        Timer.builder("event.latency")
                .tag("protocol", protocol)
                .tag("from", from)
                .tag("to", to)
                .tag("success", Boolean.toString(success))
                .register(registry)
                .record(durationNanos, TimeUnit.NANOSECONDS);
    }

    /**
     * Writes a raw event record to Chronicle Queue.
     */
    private void writeRawEvent(long requestId, String protocol, String stage, long timestamp, boolean success) {
        appender.writeDocument(w -> {
            w.write("requestId").int64(requestId)
             .write("protocol").text(protocol)
             .write("stage").text(stage)
             .write("timestamp").int64(timestamp)
             .write("success").bool(success);
        });
    }

    /**
     * Represents a single event checkpoint in a lifecycle.
     */
    private static class EventCheckpoint {
        final String name;
        final long timestamp;

        EventCheckpoint(String name, long timestamp) {
            this.name = Objects.requireNonNull(name, "Stage name cannot be null");
            this.timestamp = timestamp;
        }
    }
}
```
