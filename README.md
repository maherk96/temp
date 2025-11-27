```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.test.util.ReflectionTestUtils;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class ProcessingGuardTest {

    private ProcessingGuard guard;

    @BeforeEach
    void setUp() throws Exception {
        guard = new ProcessingGuard();
        ReflectionTestUtils.setField(guard, "hostname", "my-host");
    }

    @Test
    void whenGuardDisabled_thenAlwaysAllowProcessing() {
        ReflectionTestUtils.setField(guard, "guardEnabled", false);
        ReflectionTestUtils.setField(guard, "allowedHosts", List.of());

        assertTrue(guard.allowProcessing());
    }

    @Test
    void whenGuardEnabled_andHostnameMatches_thenAllowProcessing() {
        ReflectionTestUtils.setField(guard, "guardEnabled", true);
        ReflectionTestUtils.setField(guard, "allowedHosts", List.of("my-host"));

        assertTrue(guard.allowProcessing());
    }

    @Test
    void whenGuardEnabled_andHostnameDoesNotMatch_thenBlockProcessing() {
        ReflectionTestUtils.setField(guard, "guardEnabled", true);
        ReflectionTestUtils.setField(guard, "allowedHosts", List.of("uat-server-01"));

        assertFalse(guard.allowProcessing());
    }

    @Test
    void whenGuardEnabled_andAllowedHostsEmpty_thenBlockProcessing() {
        ReflectionTestUtils.setField(guard, "guardEnabled", true);
        ReflectionTestUtils.setField(guard, "allowedHosts", List.of());

        assertFalse(guard.allowProcessing());
    }
}


```
