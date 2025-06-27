```java
package com.taskvolt.taskvolt;

import java.util.UUID;

/**
 * Responsible for generating and managing a JVM-wide launch ID stored as a system property.
 * If a base value is already present in the system property, a UUID is appended once.
 * The result is truncated to a fixed max length.
 */
public class QAPLaunchIdGenerator {

    private static final String SYSTEM_PROPERTY_LAUNCH_ID = "launchID";
    private static final int MAX_LAUNCH_ID_LENGTH = 35;
    private static final int UUID_LENGTH = 12;

    /**
     * Generates a launch ID if one does not already exist in the expected format.
     * If the system property contains a plain base, a UUID is appended to form the full ID.
     * If the property is already in full format (base-UUID), no changes are made.
     */
    public void generateLaunchId() {
        String current = System.getProperty(SYSTEM_PROPERTY_LAUNCH_ID);

        if (isFullLaunchId(current)) {
            return;
        }

        String base = extractBase(current);
        String launchId = base + "-" + generateShortUUID();
        System.setProperty(SYSTEM_PROPERTY_LAUNCH_ID, truncate(launchId));
    }

    /**
     * Retrieves the current launch ID from the system property.
     * @return the launch ID or null if not set
     */
    public String getLaunchId() {
        return System.getProperty(SYSTEM_PROPERTY_LAUNCH_ID);
    }
    
    /**
     * Checks whether the current value looks like a complete launch ID (i.e. base-UUID).
     */
    private boolean isFullLaunchId(String value) {
        return value != null && value.matches(".+-[a-zA-Z0-9]{" + UUID_LENGTH + ",}");
    }

    /**
     * Extracts the base from the existing property or uses a default if empty.
     */
    private String extractBase(String current) {
        if (current == null || current.isBlank()) {
            return "TestLaunch";
        }
        return current.split("-")[0];
    }

    /**
     * Generates a short random UUID string with no dashes.
     */
    private String generateShortUUID() {
        return UUID.randomUUID().toString().replace("-", "").substring(0, UUID_LENGTH);
    }

    /**
     * Truncates the launch ID to the max length, removing trailing dashes if needed.
     */
    private String truncate(String launchId) {
        if (launchId.length() > MAX_LAUNCH_ID_LENGTH) {
            return launchId.substring(0, MAX_LAUNCH_ID_LENGTH).replaceAll("-+$", "");
        }
        return launchId;
    }
}

package com.taskvolt.taskvolt;

import org.junit.jupiter.api.*;

import java.util.concurrent.*;
import java.util.*;

import static org.junit.jupiter.api.Assertions.*;

public class QAPLaunchIdGeneratorTest {

    private QAPLaunchIdGenerator manager;
    private static final String SYSTEM_PROPERTY_KEY = "launchID";

    @BeforeEach
    void resetLaunchId() {
        System.clearProperty(SYSTEM_PROPERTY_KEY);
        manager = new QAPLaunchIdGenerator();
    }

    @Test
    @DisplayName("Generates default launch ID only once when no base is provided")
    void generatesDefaultLaunchIdOnce() {
        manager.generateLaunchId();
        var firstId = manager.getLaunchId();
        assertNotNull(firstId);
        assertTrue(firstId.startsWith("TestLaunch-"), "Should start with default base 'TestLaunch-'");
        assertTrue(firstId.length() <= 35, "Launch ID should be no longer than 35 characters");
        manager.generateLaunchId();
        var secondId = manager.getLaunchId();
        assertEquals(firstId, secondId, "Launch ID should not change once generated");
    }

    @Test
    @DisplayName("Appends UUID to pre-set system property base")
    void appendsUuidToPresetBase() {
        System.setProperty(SYSTEM_PROPERTY_KEY, "RegressionPack");
        manager.generateLaunchId();
        var id = manager.getLaunchId();

        assertNotNull(id);
        assertTrue(id.startsWith("RegressionPack-"), "Should start with pre-set base 'RegressionPack-'");
        assertTrue(id.length() <= 35, "Launch ID should be no longer than 35 characters");
    }

    @Test
    @DisplayName("Provides consistent launch ID across multiple threads")
    void consistentLaunchIdAcrossThreads() throws Exception {
        manager.generateLaunchId();
        var expectedId = manager.getLaunchId();

        var executor = Executors.newFixedThreadPool(5);
        List<Future<String>> results = new ArrayList<>();

        for (int i = 0; i < 5; i++) {
            results.add(executor.submit(manager::getLaunchId));
        }

        for (Future<String> future : results) {
            assertEquals(expectedId, future.get(), "All threads should see the same launch ID");
        }

        executor.shutdown();
    }

}
```
