```java
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

@SpringBootTest
@ActiveProfiles("test")
class HeatmapDataServiceMappingTest {
    
    @Autowired
    private HeatmapDataService heatmapDataService;

    @Test
    @DisplayName("1. Single class, single tag - ACTIVE status (run in period 0)")
    void testSingleClassSingleTagActive() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p0End = p0Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        rawData.add(new TestTagStatisticsData(
                "OrderTests", "SMOKE", 0,
                p0Start, p0End,
                10, 10, 9,
                p0Start.plusDays(2)
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("OrderTests");
        assertThat(dto.tag()).isEqualTo("SMOKE");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.ACTIVE);
        assertThat(dto.passed()).isEqualTo(9);
        assertThat(dto.totalTestRuns()).isEqualTo(10);
        assertThat(dto.passedRate()).isEqualTo(90.0);
        assertThat(dto.lastRun()).isEqualTo(p0Start.plusDays(2).format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm")));
    }

    @Test
    @DisplayName("2. Single class, single tag - STALE status (run only in period 1)")
    void testSingleClassSingleTagStale() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p0End = p0Start.plusWeeks(1);
        LocalDateTime p1Start = p0Start.minusWeeks(1);
        LocalDateTime p1End = p1Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        rawData.add(new TestTagStatisticsData(
                "PaymentTests", "REGRESSION", 0,
                p0Start, p0End,
                15, 0, 0,
                null // No run in period 0
        ));
        rawData.add(new TestTagStatisticsData(
                "PaymentTests", "REGRESSION", 1,
                p1Start, p1End,
                15, 12, 10,
                p1Start.plusDays(3) // Last run in period 1
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("PaymentTests");
        assertThat(dto.tag()).isEqualTo("REGRESSION");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.STALE);
        assertThat(dto.passed()).isEqualTo(10);
        assertThat(dto.totalTestRuns()).isEqualTo(12);
        assertThat(dto.lastRun()).isEqualTo(p1Start.plusDays(3).format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm")));
    }

    @Test
    @DisplayName("3. Single class, single tag - NOT_RUN status (no runs in any period)")
    void testSingleClassSingleTagNotRun() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p0End = p0Start.plusWeeks(1);
        LocalDateTime p1Start = p0Start.minusWeeks(1);
        LocalDateTime p1End = p1Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        rawData.add(new TestTagStatisticsData(
                "InventoryTests", "NIGHTLY", 0,
                p0Start, p0End,
                20, 0, 0,
                null
        ));
        rawData.add(new TestTagStatisticsData(
                "InventoryTests", "NIGHTLY", 1,
                p1Start, p1End,
                20, 0, 0,
                null
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("InventoryTests");
        assertThat(dto.tag()).isEqualTo("NIGHTLY");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.NOT_RUN);
        assertThat(dto.passed()).isEqualTo(0);
        assertThat(dto.totalTestRuns()).isEqualTo(0);
        assertThat(dto.lastRun()).contains("never");
    }

    @Test
    @DisplayName("4. One class, multiple tags - mixed statuses")
    void testOneClassMultipleTags() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p0End = p0Start.plusWeeks(1);
        LocalDateTime p1Start = p0Start.minusWeeks(1);
        LocalDateTime p1End = p1Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        // Tag UNIT - ACTIVE
        rawData.add(new TestTagStatisticsData(
                "CustomerTests", "UNIT", 0,
                p0Start, p0End,
                25, 25, 24,
                p0Start.plusDays(1)
        ));
        
        // Tag INTEGRATION - STALE
        rawData.add(new TestTagStatisticsData(
                "CustomerTests", "INTEGRATION", 0,
                p0Start, p0End,
                15, 0, 0,
                null
        ));
        rawData.add(new TestTagStatisticsData(
                "CustomerTests", "INTEGRATION", 1,
                p1Start, p1End,
                15, 10, 8,
                p1Start.plusDays(4)
        ));
        
        // Tag E2E - NOT_RUN
        rawData.add(new TestTagStatisticsData(
                "CustomerTests", "E2E", 0,
                p0Start, p0End,
                5, 0, 0,
                null
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(3);
        
        HeatmapAnalysisDTO unit = result.stream()
                .filter(dto -> dto.tag().equals("UNIT"))
                .findFirst().orElseThrow();
        assertThat(unit.status()).isEqualTo(HeatmapStatus.ACTIVE);
        assertThat(unit.passed()).isEqualTo(24);
        
        HeatmapAnalysisDTO integration = result.stream()
                .filter(dto -> dto.tag().equals("INTEGRATION"))
                .findFirst().orElseThrow();
        assertThat(integration.status()).isEqualTo(HeatmapStatus.STALE);
        assertThat(integration.passed()).isEqualTo(8);
        
        HeatmapAnalysisDTO e2e = result.stream()
                .filter(dto -> dto.tag().equals("E2E"))
                .findFirst().orElseThrow();
        assertThat(e2e.status()).isEqualTo(HeatmapStatus.NOT_RUN);
    }

    @Test
    @DisplayName("5. Multiple classes, multiple tags - all ACTIVE")
    void testMultipleClassesMultipleTagsAllActive() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p0End = p0Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        rawData.add(new TestTagStatisticsData(
                "AuthTests", "SECURITY", 0,
                p0Start, p0End,
                8, 8, 8,
                p0Start.plusDays(1)
        ));
        rawData.add(new TestTagStatisticsData(
                "AuthTests", "CRITICAL", 0,
                p0Start, p0End,
                12, 12, 11,
                p0Start.plusDays(2)
        ));
        rawData.add(new TestTagStatisticsData(
                "BillingTests", "SECURITY", 0,
                p0Start, p0End,
                6, 6, 5,
                p0Start.plusDays(3)
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(3);
        assertThat(result).allMatch(dto -> dto.status() == HeatmapStatus.ACTIVE);
        
        assertThat(result).anyMatch(dto -> 
                dto.testClassName().equals("AuthTests") && dto.tag().equals("SECURITY"));
        assertThat(result).anyMatch(dto -> 
                dto.testClassName().equals("AuthTests") && dto.tag().equals("CRITICAL"));
        assertThat(result).anyMatch(dto -> 
                dto.testClassName().equals("BillingTests") && dto.tag().equals("SECURITY"));
    }

    @Test
    @DisplayName("6. Activity only in past periods (periods 2 and 3) - STALE")
    void testActivityOnlyInPastPeriods() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p0End = p0Start.plusWeeks(1);
        LocalDateTime p1Start = p0Start.minusWeeks(1);
        LocalDateTime p1End = p1Start.plusWeeks(1);
        LocalDateTime p2Start = p0Start.minusWeeks(2);
        LocalDateTime p2End = p2Start.plusWeeks(1);
        LocalDateTime p3Start = p0Start.minusWeeks(3);
        LocalDateTime p3End = p3Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        rawData.add(new TestTagStatisticsData(
                "LegacyTests", "DEPRECATED", 0,
                p0Start, p0End,
                10, 0, 0,
                null
        ));
        rawData.add(new TestTagStatisticsData(
                "LegacyTests", "DEPRECATED", 1,
                p1Start, p1End,
                10, 0, 0,
                null
        ));
        rawData.add(new TestTagStatisticsData(
                "LegacyTests", "DEPRECATED", 2,
                p2Start, p2End,
                10, 8, 6,
                p2Start.plusDays(2)
        ));
        rawData.add(new TestTagStatisticsData(
                "LegacyTests", "DEPRECATED", 3,
                p3Start, p3End,
                10, 5, 4,
                p3Start.plusDays(1)
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("LegacyTests");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.STALE);
        assertThat(dto.lastRun()).isEqualTo(p2Start.plusDays(2).format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm")));
        assertThat(dto.passed()).isEqualTo(6);
    }

    @Test
    @DisplayName("7. Activity in both past and present periods - ACTIVE (chooses latest)")
    void testActivityInPastAndPresent() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p0End = p0Start.plusWeeks(1);
        LocalDateTime p1Start = p0Start.minusWeeks(1);
        LocalDateTime p1End = p1Start.plusWeeks(1);
        LocalDateTime p2Start = p0Start.minusWeeks(2);
        LocalDateTime p2End = p2Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        rawData.add(new TestTagStatisticsData(
                "ContinuousTests", "MONITORING", 0,
                p0Start, p0End,
                30, 28, 27,
                p0Start.plusDays(5) // Latest run
        ));
        rawData.add(new TestTagStatisticsData(
                "ContinuousTests", "MONITORING", 1,
                p1Start, p1End,
                30, 25, 23,
                p1Start.plusDays(4)
        ));
        rawData.add(new TestTagStatisticsData(
                "ContinuousTests", "MONITORING", 2,
                p2Start, p2End,
                30, 20, 18,
                p2Start.plusDays(2)
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("ContinuousTests");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.ACTIVE);
        assertThat(dto.lastRun()).isEqualTo(p0Start.plusDays(5).format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm")));
        assertThat(dto.passed()).isEqualTo(27);
        assertThat(dto.totalTestRuns()).isEqualTo(28);
    }

    @Test
    @DisplayName("8. Null or empty raw data list")
    void testNullOrEmptyRawData() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();

        // Test with empty list
        List<TestTagStatisticsData> emptyData = new ArrayList<>();
        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(emptyData, p0Start);

        assertThat(result).isEmpty();

        // Test with null (if your service handles it)
        // Uncomment if your service should handle null gracefully
        // List<HeatmapAnalysisDTO> nullResult = 
        //         heatmapDataService.buildHeatmapResponse(null, p0Start);
        // assertThat(nullResult).isEmpty();
    }

    @Test
    @DisplayName("9. Raw data containing only prior periods (no period 0)")
    void testOnlyPriorPeriods() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p1Start = p0Start.minusWeeks(1);
        LocalDateTime p1End = p1Start.plusWeeks(1);
        LocalDateTime p2Start = p0Start.minusWeeks(2);
        LocalDateTime p2End = p2Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        // Only periods 1 and 2, no period 0
        rawData.add(new TestTagStatisticsData(
                "HistoricalTests", "ARCHIVE", 1,
                p1Start, p1End,
                12, 10, 9,
                p1Start.plusDays(3)
        ));
        rawData.add(new TestTagStatisticsData(
                "HistoricalTests", "ARCHIVE", 2,
                p2Start, p2End,
                12, 8, 7,
                p2Start.plusDays(2)
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("HistoricalTests");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.STALE);
        assertThat(dto.lastRun()).isEqualTo(p1Start.plusDays(3).format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm")));
    }

    @Test
    @DisplayName("10. Raw data containing only period 0")
    void testOnlyPeriodZero() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p0End = p0Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        rawData.add(new TestTagStatisticsData(
                "NewTests", "BETA", 0,
                p0Start, p0End,
                5, 5, 4,
                p0Start.plusDays(1)
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("NewTests");
        assertThat(dto.tag()).isEqualTo("BETA");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.ACTIVE);
        assertThat(dto.passed()).isEqualTo(4);
        assertThat(dto.totalTestRuns()).isEqualTo(5);
    }

    @Test
    @DisplayName("11. Requested period entirely after latest run - STALE")
    void testRequestedPeriodAfterLatestRun() {
        // Requested period is in the future relative to all test runs
        LocalDateTime p0Start = LocalDateTime.of(2025, 12, 1, 0, 0);
        LocalDateTime p0End = p0Start.plusWeeks(1);
        LocalDateTime p1Start = p0Start.minusWeeks(1);
        LocalDateTime p1End = p1Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        rawData.add(new TestTagStatisticsData(
                "OldTests", "UNMAINTAINED", 0,
                p0Start, p0End,
                10, 0, 0,
                null // No runs in requested period
        ));
        rawData.add(new TestTagStatisticsData(
                "OldTests", "UNMAINTAINED", 1,
                p1Start, p1End,
                10, 5, 4,
                LocalDateTime.of(2025, 11, 25, 10, 0) // Last run before p0
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("OldTests");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.STALE);
        assertThat(dto.lastRun()).isEqualTo(LocalDateTime.of(2025, 11, 25, 10, 0).format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm")));
    }

    @Test
    @DisplayName("12. Requested period with multiple runs - should choose latest")
    void testMultipleRunsChooseLatest() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p0End = p0Start.plusWeeks(1);
        LocalDateTime p1Start = p0Start.minusWeeks(1);
        LocalDateTime p1End = p1Start.plusWeeks(1);
        LocalDateTime p2Start = p0Start.minusWeeks(2);
        LocalDateTime p2End = p2Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        // Multiple runs across periods - p0 has latest
        rawData.add(new TestTagStatisticsData(
                "FrequentTests", "HOURLY", 0,
                p0Start, p0End,
                50, 45, 43,
                p0Start.plusDays(4).plusHours(15) // Latest
        ));
        rawData.add(new TestTagStatisticsData(
                "FrequentTests", "HOURLY", 1,
                p1Start, p1End,
                50, 40, 38,
                p1Start.plusDays(3)
        ));
        rawData.add(new TestTagStatisticsData(
                "FrequentTests", "HOURLY", 2,
                p2Start, p2End,
                50, 35, 32,
                p2Start.plusDays(2)
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("FrequentTests");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.ACTIVE);
        assertThat(dto.lastRun()).isEqualTo(p0Start.plusDays(4).plusHours(15).format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm")));
        assertThat(dto.passed()).isEqualTo(43);
        assertThat(dto.totalTestRuns()).isEqualTo(45);
    }

    @Test
    @DisplayName("13. Invalid data - testsRun = 0 but testsPassed > 0")
    void testInvalidDataTestsRunZeroPassedNonZero() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p0End = p0Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        // Invalid: 0 tests run but 5 passed
        rawData.add(new TestTagStatisticsData(
                "InvalidTests", "DATA_ERROR", 0,
                p0Start, p0End,
                10, 0, 5, // testsWithRunInPeriod=0, passedTests=5 (invalid)
                p0Start.plusDays(1)
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("InvalidTests");
        // Service should handle gracefully - passedRate should be 0 or handle error
        assertThat(dto.totalTestRuns()).isEqualTo(0);
        // The service's getPassRate() returns 0 when testsWithRunInPeriod is 0
        assertThat(dto.passedRate()).isEqualTo(0.0);
    }

    @Test
    @DisplayName("14. Week-aligned requested period boundaries")
    void testWeekAlignedPeriodBoundaries() {
        // Monday start, exactly one week duration
        LocalDateTime p0Start = LocalDateTime.of(2025, 11, 17, 0, 0); // Monday
        LocalDateTime p0End = LocalDateTime.of(2025, 11, 24, 0, 0); // Next Monday
        LocalDateTime p1Start = p0Start.minusWeeks(1);
        LocalDateTime p1End = p1Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        rawData.add(new TestTagStatisticsData(
                "WeeklyTests", "SCHEDULED", 0,
                p0Start, p0End,
                14, 14, 13,
                p0Start.plusDays(3)
        ));
        rawData.add(new TestTagStatisticsData(
                "WeeklyTests", "SCHEDULED", 1,
                p1Start, p1End,
                14, 12, 11,
                p1Start.plusDays(2)
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("WeeklyTests");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.ACTIVE);
        assertThat(dto.lastRun()).isEqualTo(p0Start.plusDays(3).format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm")));
        assertThat(dto.passed()).isEqualTo(13);
        assertThat(dto.totalTestRuns()).isEqualTo(14);
        assertThat(dto.passedRate()).isCloseTo(92.86, within(0.01));
    }


@Test
    @DisplayName("15. Multiple entries for same class/tag in same period - should choose latest run")
    void testMultipleEntriesForSameClassTagInSamePeriod() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p0End = p0Start.plusWeeks(1);
        LocalDateTime p1Start = p0Start.minusWeeks(1);
        LocalDateTime p1End = p1Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        // Two entries for same class/tag in period 0, different run times
        rawData.add(new TestTagStatisticsData(
                "PaymentTests", "CRITICAL", 0,
                p0Start, p0End,
                10, 8, 8,
                p0Start.plusDays(1) // Earlier run
        ));
        rawData.add(new TestTagStatisticsData(
                "PaymentTests", "CRITICAL", 0,
                p0Start, p0End,
                10, 10, 9,
                p0Start.plusDays(3) // Later run - should be chosen
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("PaymentTests");
        assertThat(dto.tag()).isEqualTo("CRITICAL");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.ACTIVE);
        assertThat(dto.passed()).isEqualTo(9); // From later entry
        assertThat(dto.totalTestRuns()).isEqualTo(10); // From later entry
        assertThat(dto.lastRun()).isEqualTo(p0Start.plusDays(3).format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm")));
    }

    @Test
    @DisplayName("16. Out-of-order period indices")
    void testOutOfOrderPeriodIndices() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p0End = p0Start.plusWeeks(1);
        LocalDateTime p1Start = p0Start.minusWeeks(1);
        LocalDateTime p1End = p1Start.plusWeeks(1);
        LocalDateTime p2Start = p0Start.minusWeeks(2);
        LocalDateTime p2End = p2Start.plusWeeks(1);
        LocalDateTime p3Start = p0Start.minusWeeks(3);
        LocalDateTime p3End = p3Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        // Add in reverse order
        rawData.add(new TestTagStatisticsData(
                "SecurityTests", "AUTH", 3,
                p3Start, p3End,
                5, 4, 4,
                p3Start.plusDays(1)
        ));
        rawData.add(new TestTagStatisticsData(
                "SecurityTests", "AUTH", 1,
                p1Start, p1End,
                5, 5, 4,
                p1Start.plusDays(2)
        ));
        rawData.add(new TestTagStatisticsData(
                "SecurityTests", "AUTH", 0,
                p0Start, p0End,
                5, 5, 5,
                p0Start.plusDays(3)
        ));
        rawData.add(new TestTagStatisticsData(
                "SecurityTests", "AUTH", 2,
                p2Start, p2End,
                5, 3, 3,
                p2Start.plusDays(4)
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("SecurityTests");
        assertThat(dto.tag()).isEqualTo("AUTH");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.ACTIVE);
        assertThat(dto.lastRun()).isEqualTo(p0Start.plusDays(3).format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm")));
        assertThat(dto.passed()).isEqualTo(5);
        assertThat(dto.totalTestRuns()).isEqualTo(5);
    }

    @Test
    @DisplayName("17. Future lastRunDateTime relative to system clock")
    void testFutureLastRunDateTime() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p0End = p0Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        // Last run is in the future
        rawData.add(new TestTagStatisticsData(
                "FutureTests", "SCHEDULED", 0,
                p0Start, p0End,
                3, 3, 3,
                now.plusDays(5) // Future date
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("FutureTests");
        assertThat(dto.tag()).isEqualTo("SCHEDULED");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.ACTIVE);
        assertThat(dto.lastRun()).isEqualTo(now.plusDays(5).format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm")));
    }

    @Test
    @DisplayName("18. LastRunDateTime equals period endDateTime")
    void testLastRunEqualsEndDateTime() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p0End = p0Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        // Last run exactly at period end
        rawData.add(new TestTagStatisticsData(
                "BoundaryTests", "EDGE", 0,
                p0Start, p0End,
                7, 7, 6,
                p0End // Exactly at end
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("BoundaryTests");
        assertThat(dto.tag()).isEqualTo("EDGE");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.ACTIVE);
        assertThat(dto.lastRun()).isEqualTo(p0End.format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm")));
        assertThat(dto.passed()).isEqualTo(6);
        assertThat(dto.totalTestRuns()).isEqualTo(7);
    }

    @Test
    @DisplayName("19. Two-week requested range")
    void testTwoWeekRequestedRange() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p0End = p0Start.plusWeeks(2); // Two weeks
        LocalDateTime p1Start = p0Start.minusWeeks(2);
        LocalDateTime p1End = p1Start.plusWeeks(2);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        rawData.add(new TestTagStatisticsData(
                "LongRangeTests", "BIWEEKLY", 0,
                p0Start, p0End,
                40, 35, 30,
                p0Start.plusDays(10)
        ));
        rawData.add(new TestTagStatisticsData(
                "LongRangeTests", "BIWEEKLY", 1,
                p1Start, p1End,
                40, 20, 15,
                p1Start.plusDays(5)
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("LongRangeTests");
        assertThat(dto.tag()).isEqualTo("BIWEEKLY");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.ACTIVE);
        assertThat(dto.lastRun()).isEqualTo(p0Start.plusDays(10).format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm")));
        assertThat(dto.passed()).isEqualTo(30);
        assertThat(dto.totalTestRuns()).isEqualTo(35);
    }

    @Test
    @DisplayName("20. Three-week requested range")
    void testThreeWeekRequestedRange() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p0End = p0Start.plusWeeks(3); // Three weeks
        LocalDateTime p1Start = p0Start.minusWeeks(3);
        LocalDateTime p1End = p1Start.plusWeeks(3);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        rawData.add(new TestTagStatisticsData(
                "MonthlyTests", "COMPREHENSIVE", 0,
                p0Start, p0End,
                100, 90, 85,
                p0Start.plusDays(15)
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("MonthlyTests");
        assertThat(dto.tag()).isEqualTo("COMPREHENSIVE");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.ACTIVE);
        assertThat(dto.lastRun()).isEqualTo(p0Start.plusDays(15).format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm")));
        assertThat(dto.passed()).isEqualTo(85);
        assertThat(dto.totalTestRuns()).isEqualTo(90);
        assertThat(dto.passedRate()).isCloseTo(94.44, within(0.01));
    }

    @Test
    @DisplayName("21. Requested period not aligned with week boundaries - mid-week start")
    void testRequestedPeriodMidWeekStart() {
        // Start on a Wednesday instead of Monday
        LocalDateTime p0Start = LocalDateTime.of(2025, 11, 19, 0, 0); // Wednesday
        LocalDateTime p0End = p0Start.plusWeeks(1);
        LocalDateTime p1Start = p0Start.minusWeeks(1);
        LocalDateTime p1End = p1Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        rawData.add(new TestTagStatisticsData(
                "MidWeekTests", "FLEXIBLE", 0,
                p0Start, p0End,
                12, 10, 9,
                p0Start.plusDays(2)
        ));
        rawData.add(new TestTagStatisticsData(
                "MidWeekTests", "FLEXIBLE", 1,
                p1Start, p1End,
                12, 8, 7,
                p1Start.plusDays(3)
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("MidWeekTests");
        assertThat(dto.tag()).isEqualTo("FLEXIBLE");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.ACTIVE);
        assertThat(dto.lastRun()).isEqualTo(p0Start.plusDays(2).format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm")));
        assertThat(dto.passed()).isEqualTo(9);
    }

    @Test
    @DisplayName("22. Complex scenario - Multiple classes, multiple tags, mixed statuses")
    void testComplexMultiClassMultiTagMixedStatuses() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p0End = p0Start.plusWeeks(1);
        LocalDateTime p1Start = p0Start.minusWeeks(1);
        LocalDateTime p1End = p1Start.plusWeeks(1);
        LocalDateTime p2Start = p0Start.minusWeeks(2);
        LocalDateTime p2End = p2Start.plusWeeks(1);
        LocalDateTime p3Start = p0Start.minusWeeks(3);
        LocalDateTime p3End = p3Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        // Class A, Tag X - ACTIVE (run in period 0)
        rawData.add(new TestTagStatisticsData(
                "ClassA", "TagX", 0,
                p0Start, p0End,
                10, 10, 9,
                p0Start.plusDays(2)
        ));
        
        // Class A, Tag Y - STALE (only run in period 2)
        rawData.add(new TestTagStatisticsData(
                "ClassA", "TagY", 2,
                p2Start, p2End,
                8, 6, 5,
                p2Start.plusDays(1)
        ));
        
        // Class B, Tag X - NOT_RUN (no runs in any period)
        rawData.add(new TestTagStatisticsData(
                "ClassB", "TagX", 0,
                p0Start, p0End,
                15, 0, 0,
                null
        ));
        
        // Class B, Tag Z - ACTIVE (run in period 0)
        rawData.add(new TestTagStatisticsData(
                "ClassB", "TagZ", 0,
                p0Start, p0End,
                20, 18, 17,
                p0Start.plusDays(4)
        ));
        rawData.add(new TestTagStatisticsData(
                "ClassB", "TagZ", 1,
                p1Start, p1End,
                20, 15, 14,
                p1Start.plusDays(2)
        ));
        
        // Class C, Tag W - STALE (run in period 1 and 3, but not 0)
        rawData.add(new TestTagStatisticsData(
                "ClassC", "TagW", 1,
                p1Start, p1End,
                12, 10, 9,
                p1Start.plusDays(3)
        ));
        rawData.add(new TestTagStatisticsData(
                "ClassC", "TagW", 3,
                p3Start, p3End,
                12, 8, 7,
                p3Start.plusDays(1)
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(5);
        
        // Find each DTO and verify
        HeatmapAnalysisDTO classATagX = result.stream()
                .filter(dto -> dto.testClassName().equals("ClassA") && dto.tag().equals("TagX"))
                .findFirst().orElseThrow();
        assertThat(classATagX.status()).isEqualTo(HeatmapStatus.ACTIVE);
        assertThat(classATagX.passed()).isEqualTo(9);
        assertThat(classATagX.totalTestRuns()).isEqualTo(10);
        
        HeatmapAnalysisDTO classATagY = result.stream()
                .filter(dto -> dto.testClassName().equals("ClassA") && dto.tag().equals("TagY"))
                .findFirst().orElseThrow();
        assertThat(classATagY.status()).isEqualTo(HeatmapStatus.STALE);
        assertThat(classATagY.passed()).isEqualTo(5);
        
        HeatmapAnalysisDTO classBTagX = result.stream()
                .filter(dto -> dto.testClassName().equals("ClassB") && dto.tag().equals("TagX"))
                .findFirst().orElseThrow();
        assertThat(classBTagX.status()).isEqualTo(HeatmapStatus.NOT_RUN);
        assertThat(classBTagX.lastRun()).contains("never");
        
        HeatmapAnalysisDTO classBTagZ = result.stream()
                .filter(dto -> dto.testClassName().equals("ClassB") && dto.tag().equals("TagZ"))
                .findFirst().orElseThrow();
        assertThat(classBTagZ.status()).isEqualTo(HeatmapStatus.ACTIVE);
        assertThat(classBTagZ.passed()).isEqualTo(17);
        
        HeatmapAnalysisDTO classCTagW = result.stream()
                .filter(dto -> dto.testClassName().equals("ClassC") && dto.tag().equals("TagW"))
                .findFirst().orElseThrow();
        assertThat(classCTagW.status()).isEqualTo(HeatmapStatus.STALE);
        assertThat(classCTagW.lastRun()).isEqualTo(p1Start.plusDays(3).format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm")));
    }

    @Test
    @DisplayName("23. Requested period before earliest run - all STALE")
    void testRequestedPeriodBeforeEarliestRun() {
        // Set requested period in the past
        LocalDateTime p0Start = LocalDateTime.of(2025, 10, 1, 0, 0);
        LocalDateTime p0End = p0Start.plusWeeks(1);
        
        // All runs happen AFTER the requested period
        LocalDateTime futureRunTime = LocalDateTime.of(2025, 11, 15, 10, 0);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        rawData.add(new TestTagStatisticsData(
                "FutureOnlyTests", "DELAYED", 0,
                p0Start, p0End,
                5, 0, 0,
                null // No run in period 0
        ));
        rawData.add(new TestTagStatisticsData(
                "FutureOnlyTests", "DELAYED", 1,
                p0Start.minusWeeks(1), p0Start,
                5, 0, 0,
                null
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("FutureOnlyTests");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.NOT_RUN);
    }

    @Test
    @DisplayName("24. Period boundaries overlap - should handle gracefully")
    void testOverlappingPeriodBoundaries() {
        LocalDateTime p0Start = LocalDateTime.of(2025, 11, 17, 0, 0);
        LocalDateTime p0End = LocalDateTime.of(2025, 11, 24, 0, 0);
        
        // Create overlapping period 1
        LocalDateTime p1Start = LocalDateTime.of(2025, 11, 10, 0, 0);
        LocalDateTime p1End = LocalDateTime.of(2025, 11, 20, 0, 0); // Overlaps with p0

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        rawData.add(new TestTagStatisticsData(
                "OverlapTests", "BOUNDARY", 0,
                p0Start, p0End,
                10, 8, 7,
                LocalDateTime.of(2025, 11, 18, 10, 0)
        ));
        rawData.add(new TestTagStatisticsData(
                "OverlapTests", "BOUNDARY", 1,
                p1Start, p1End,
                10, 5, 4,
                LocalDateTime.of(2025, 11, 12, 14, 0)
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.testClassName()).isEqualTo("OverlapTests");
        assertThat(dto.status()).isEqualTo(HeatmapStatus.ACTIVE);
        assertThat(dto.lastRun()).isEqualTo(LocalDateTime.of(2025, 11, 18, 10, 0).format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm")));
    }

    @Test
    @DisplayName("25. Verify period analysis message format")
    void testPeriodAnalysisMessageFormat() {
        LocalDateTime now = LocalDateTime.of(2025, 11, 20, 14, 0);
        LocalDate weekStart = now.toLocalDate().with(java.time.DayOfWeek.MONDAY);
        LocalDateTime p0Start = weekStart.atStartOfDay();
        LocalDateTime p0End = p0Start.plusWeeks(1);
        LocalDateTime p1Start = p0Start.minusWeeks(1);
        LocalDateTime p1End = p1Start.plusWeeks(1);
        LocalDateTime p2Start = p0Start.minusWeeks(2);
        LocalDateTime p2End = p2Start.plusWeeks(1);

        List<TestTagStatisticsData> rawData = new ArrayList<>();
        
        // Period 0 - should contain "selected period"
        rawData.add(new TestTagStatisticsData(
                "MessageTests", "FORMAT", 0,
                p0Start, p0End,
                5, 5, 5,
                p0Start.plusDays(1)
        ));

        List<HeatmapAnalysisDTO> result = 
                heatmapDataService.buildHeatmapResponse(rawData, p0Start);

        assertThat(result).hasSize(1);
        HeatmapAnalysisDTO dto = result.get(0);
        assertThat(dto.periodAnalysis()).contains("selected period");
        assertThat(dto.periodAnalysis()).contains(p0Start.format(DateTimeFormatter.ofPattern("dd MMM yyyy")));
    }
}
```
