# temp

```java
package com.taskvolt.taskvolt;

import java.util.UUID;

/**
 * Utility class for handling extension-related operations.
 * Thread-safe. Generates consistent launch IDs with format base-UUID.
 */
public class ExtensionUtil {
    public static final String SYSTEM_PROPERTY_LAUNCH_ID = "launchID";
    public static final int MAX_LAUNCH_ID_LENGTH = 35;

    // Thread-local base tracking to avoid collision across threads
    private static final ThreadLocal<String> lastBase = new ThreadLocal<>();

    public static void main(String[] args) {
        setLaunchId(null); // TestLaunch-UUID
        System.out.println("Launch ID: " + getLaunchId());

        setLaunchId(null); // Same as above
        System.out.println("Launch ID: " + getLaunchId());

        setLaunchId("regression123"); // regression123-UUID
        System.out.println("Launch ID: " + getLaunchId());

        setLaunchId(null); // new TestLaunch-UUID
        System.out.println("Launch ID: " + getLaunchId());

        setLaunchId("regression123"); // new regression123-UUID
        System.out.println("Launch ID: " + getLaunchId());

        setLaunchId("regression123"); // same as above
        System.out.println("Launch ID: " + getLaunchId());
    }

    /**
     * Sets the launch ID based on the given base name.
     * If base changes (per thread), regenerates a new ID.
     */
    public static void setLaunchId(String customBase) {
        String base = (customBase != null && !customBase.isBlank()) ? customBase : "TestLaunch";
        String current = System.getProperty(SYSTEM_PROPERTY_LAUNCH_ID);
        String previousBase = lastBase.get();

        if (current == null || !base.equals(previousBase)) {
            String launchId = base + "-" + generateShortUUID();
            System.setProperty(SYSTEM_PROPERTY_LAUNCH_ID, truncateLaunchId(launchId));
            lastBase.set(base);
        }
        // else: reuse existing launch ID
    }

    /**
     * Retrieves the current launch ID from system properties.
     */
    public static String getLaunchId() {
        return System.getProperty(SYSTEM_PROPERTY_LAUNCH_ID);
    }

    /**
     * Truncates the launch ID if it exceeds max length.
     */
    private static String truncateLaunchId(String launchId) {
        if (launchId.length() > MAX_LAUNCH_ID_LENGTH) {
            return launchId.substring(0, MAX_LAUNCH_ID_LENGTH);
        }
        return launchId;
    }

    /**
     * Generates a shorter UUID-based string for uniqueness.
     */
    private static String generateShortUUID() {
        return UUID.randomUUID().toString().replace("-", "").substring(0, 12);
    }
}

package com.taskvolt.taskvolt;

import org.junit.jupiter.api.*;

import java.util.*;
import java.util.concurrent.*;

import static org.junit.jupiter.api.Assertions.*;

class ExtensionUtilTest {

    @BeforeEach
    void resetLaunchId() {
        System.clearProperty(ExtensionUtil.SYSTEM_PROPERTY_LAUNCH_ID);
    }

    @Test
    @DisplayName("Should generate a launch ID with default base")
    void testDefaultLaunchIdGeneration() {
        ExtensionUtil.setLaunchId(null);
        String id = ExtensionUtil.getLaunchId();

        assertNotNull(id);
        assertTrue(id.startsWith("TestLaunch-"));
        assertTrue(id.length() <= ExtensionUtil.MAX_LAUNCH_ID_LENGTH);
    }

    @Test
    @DisplayName("Should reuse the same launch ID within same base")
    void testSameLaunchIdReused() {
        ExtensionUtil.setLaunchId(null);
        String id1 = ExtensionUtil.getLaunchId();

        ExtensionUtil.setLaunchId(null);
        String id2 = ExtensionUtil.getLaunchId();

        assertEquals(id1, id2);
    }

    @Test
    @DisplayName("Should generate new launch ID if base changes")
    void testLaunchIdChangesWhenBaseChanges() {
        ExtensionUtil.setLaunchId("regression123");
        String id1 = ExtensionUtil.getLaunchId();

        ExtensionUtil.setLaunchId("performance");
        String id2 = ExtensionUtil.getLaunchId();

        assertNotEquals(id1, id2);
        assertTrue(id1.startsWith("regression123-"));
        assertTrue(id2.startsWith("performance-"));
    }

    @Test
    @DisplayName("Should reuse custom base launch ID across multiple calls")
    void testSameCustomLaunchIdReused() {
        ExtensionUtil.setLaunchId("regression123");
        String id1 = ExtensionUtil.getLaunchId();

        ExtensionUtil.setLaunchId("regression123");
        String id2 = ExtensionUtil.getLaunchId();

        assertEquals(id1, id2);
    }

    @Test
    @DisplayName("Should truncate long launch IDs")
    void testLaunchIdTruncation() {
        String longBase = "VeryLongCustomBaseNameExceedingLength";
        ExtensionUtil.setLaunchId(longBase);
        String launchId = ExtensionUtil.getLaunchId();

        assertTrue(launchId.length() <= ExtensionUtil.MAX_LAUNCH_ID_LENGTH);
    }

    @Test
    @DisplayName("Same thread and base should yield same launch ID")
    void testSameThreadGetsSameLaunchId() {
        String base = "threadTest";

        ExtensionUtil.setLaunchId(base);
        String id1 = ExtensionUtil.getLaunchId();

        ExtensionUtil.setLaunchId(base);
        String id2 = ExtensionUtil.getLaunchId();

        assertEquals(id1, id2);
    }

    @Test
    @DisplayName("Different bases in same thread should yield different launch IDs")
    void testDifferentBasesInSameThreadProduceDifferentIds() {
        ExtensionUtil.setLaunchId("baseA");
        String id1 = ExtensionUtil.getLaunchId();

        ExtensionUtil.setLaunchId("baseB");
        String id2 = ExtensionUtil.getLaunchId();

        assertNotEquals(id1, id2);
    }

    @Test
    @DisplayName("Each thread should get its own launch ID for same base")
    void testLaunchIdIsolationAcrossThreads() throws InterruptedException, ExecutionException {
        int threadCount = 5;
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        List<Future<String>> futures = new ArrayList<>();

        String base = "parallelTest";

        for (int i = 0; i < threadCount; i++) {
            futures.add(executor.submit(() -> {
                ExtensionUtil.setLaunchId(base);
                return ExtensionUtil.getLaunchId();
            }));
        }

        Set<String> launchIds = new HashSet<>();
        for (Future<String> future : futures) {
            launchIds.add(future.get());
        }

        executor.shutdown();

        // Thread-local should trigger new ID in each thread
        assertEquals(threadCount, launchIds.size(), "Each thread should get a different launch ID for same base");
    }
}
```
