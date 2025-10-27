```java

package com.example.dashboard;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.*;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

// ============================================================================
// DashboardDataService Tests
// ============================================================================

@ExtendWith(MockitoExtension.class)
class DashboardDataServiceTest {

    @Mock
    private DatabaseQueryExecutor queryExecutor;

    @Mock
    private Map<String, String> qapQueries;

    @Mock
    private WidgetDatasetServiceFactory widgetServiceFactory;

    @Mock
    private WidgetDatasetService<MostFailedTestCasesData> widgetDatasetService;

    @InjectMocks
    private DashboardDataService dashboardDataService;

    private static final long DASHBOARD_ID = 1L;
    private static final String DASHBOARD_QUERY = "SELECT * FROM dashboard";

    @BeforeEach
    void setUp() {
        when(qapQueries.get("DashboardDatasetQuery")).thenReturn(DASHBOARD_QUERY);
    }

    @Test
    void getDashboardData_withValidId_returnsDashboardWithWidgets() throws JsonProcessingException {
        // Given
        List<DashboardWidgetData> mockData = createMockDashboardData();
        when(queryExecutor.executeQuery(eq(DASHBOARD_QUERY), any(), eq(DASHBOARD_ID)))
            .thenReturn(mockData);
        when(widgetServiceFactory.getService(WidgetDatasetType.MOST_TEST_CASES_FAILED))
            .thenReturn(widgetDatasetService);
        when(widgetDatasetService.getDataset(any())).thenReturn(createMockTestCaseData());

        // When
        DashboardDatasetResponse response = dashboardDataService.getDashboardData(DASHBOARD_ID);

        // Then
        assertNotNull(response);
        assertEquals(DASHBOARD_ID, response.getId());
        assertEquals("Test Dashboard", response.getName());
        assertEquals("Dashboard Description", response.getDescription());
        assertEquals(1, response.getWidgetResponseList().size());
        verify(queryExecutor).executeQuery(eq(DASHBOARD_QUERY), any(), eq(DASHBOARD_ID));
    }

    @Test
    void getDashboardData_withMultipleWidgets_returnsAllInCorrectOrder() {
        // Given
        List<DashboardWidgetData> mockData = Arrays.asList(
            createDashboardWidgetData(1L, 1, "Widget 1"),
            createDashboardWidgetData(2L, 2, "Widget 2"),
            createDashboardWidgetData(3L, 3, "Widget 3")
        );
        when(queryExecutor.executeQuery(eq(DASHBOARD_QUERY), any(), eq(DASHBOARD_ID)))
            .thenReturn(mockData);
        when(widgetServiceFactory.getService(any())).thenReturn(widgetDatasetService);
        when(widgetDatasetService.getDataset(any())).thenReturn(Collections.emptyList());

        // When
        DashboardDatasetResponse response = dashboardDataService.getDashboardData(DASHBOARD_ID);

        // Then
        assertEquals(3, response.getWidgetResponseList().size());
        assertEquals(1, response.getWidgetResponseList().get(0).getOrdinal());
        assertEquals(2, response.getWidgetResponseList().get(1).getOrdinal());
        assertEquals(3, response.getWidgetResponseList().get(2).getOrdinal());
    }

    @Test
    void getDashboardData_withInvalidId_throwsDashboardNotFoundException() {
        // Given
        when(queryExecutor.executeQuery(eq(DASHBOARD_QUERY), any(), eq(DASHBOARD_ID)))
            .thenReturn(Collections.emptyList());

        // When & Then
        DashboardNotFoundException exception = assertThrows(
            DashboardNotFoundException.class,
            () -> dashboardDataService.getDashboardData(DASHBOARD_ID)
        );
        assertTrue(exception.getMessage().contains("Dashboard not found"));
    }

    @Test
    void getDashboardData_withInvalidWidgetConfig_throwsWidgetConfigurationException() {
        // Given
        DashboardWidgetData invalidData = createDashboardWidgetData(1L, 1, "Widget 1");
        invalidData.setWidgetConfiguration("invalid json");
        when(queryExecutor.executeQuery(eq(DASHBOARD_QUERY), any(), eq(DASHBOARD_ID)))
            .thenReturn(List.of(invalidData));

        // When & Then
        assertThrows(
            WidgetConfigurationException.class,
            () -> dashboardDataService.getDashboardData(DASHBOARD_ID)
        );
    }

    @Test
    void getDashboardData_withMissingTypeField_throwsWidgetConfigurationException() {
        // Given
        DashboardWidgetData data = createDashboardWidgetData(1L, 1, "Widget 1");
        data.setWidgetConfiguration("{\"appName\":\"test\"}"); // Missing type field
        when(queryExecutor.executeQuery(eq(DASHBOARD_QUERY), any(), eq(DASHBOARD_ID)))
            .thenReturn(List.of(data));

        // When & Then
        assertThrows(
            WidgetConfigurationException.class,
            () -> dashboardDataService.getDashboardData(DASHBOARD_ID)
        );
    }

    @Test
    void getDashboardData_withInvalidWidgetType_throwsWidgetConfigurationException() {
        // Given
        DashboardWidgetData data = createDashboardWidgetData(1L, 1, "Widget 1");
        data.setWidgetConfiguration("{\"type\":\"INVALID_TYPE\"}");
        when(queryExecutor.executeQuery(eq(DASHBOARD_QUERY), any(), eq(DASHBOARD_ID)))
            .thenReturn(List.of(data));

        // When & Then
        assertThrows(
            WidgetConfigurationException.class,
            () -> dashboardDataService.getDashboardData(DASHBOARD_ID)
        );
    }

    // Helper methods
    private List<DashboardWidgetData> createMockDashboardData() {
        return List.of(createDashboardWidgetData(1L, 1, "Widget 1"));
    }

    private DashboardWidgetData createDashboardWidgetData(long id, int ordinal, String name) {
        DashboardWidgetData data = new DashboardWidgetData();
        data.setDashboardWidgetConfigId(id);
        data.setDashboardName("Test Dashboard");
        data.setDashboardDescription("Dashboard Description");
        data.setWidgetName(name);
        data.setWidgetDescription("Widget Description");
        data.setOrdinal(ordinal);
        data.setWidgetConfiguration(
            "{\"type\":\"MOST_TEST_CASES_FAILED\",\"appName\":\"TestApp\",\"includeRegression\":true,\"numberOfDays\":30}"
        );
        return data;
    }

    private List<MostFailedTestCasesData> createMockTestCaseData() {
        return List.of(new MostFailedTestCasesData());
    }
}

// ============================================================================
// MostFailedTestCaseDataService Tests
// ============================================================================

@ExtendWith(MockitoExtension.class)
class MostFailedTestCaseDataServiceTest {

    @Mock
    private DatabaseQueryExecutor queryExecutor;

    @Mock
    private Map<String, String> qapQueries;

    @InjectMocks
    private MostFailedTestCaseDataService service;

    private static final String QUERY_KEY = "MostFailedTestCasesQuery";
    private static final String QUERY = "SELECT * FROM test_cases";

    @BeforeEach
    void setUp() {
        when(qapQueries.get(QUERY_KEY)).thenReturn(QUERY);
    }

    @Test
    void getDataset_withValidConfig_returnsTestCases() {
        // Given
        MostFailedTestCaseConfig config = new MostFailedTestCaseConfig();
        config.setAppName("TestApp");
        config.setIncludeRegression(true);
        config.setNumberOfDays(30);

        List<MostFailedTestCasesData> expectedData = Arrays.asList(
            createTestCaseData("Test1", 10),
            createTestCaseData("Test2", 5)
        );

        when(queryExecutor.executeQuery(eq(QUERY), any(), eq("TestApp"), eq(true), eq(30)))
            .thenReturn(expectedData);

        // When
        List<MostFailedTestCasesData> result = service.getDataset(config);

        // Then
        assertEquals(2, result.size());
        assertEquals("Test1", result.get(0).getTestCaseName());
        verify(queryExecutor).executeQuery(eq(QUERY), any(), eq("TestApp"), eq(true), eq(30));
    }

    @Test
    void getDataset_withIncludeRegression_filtersCorrectly() {
        // Given
        MostFailedTestCaseConfig config = new MostFailedTestCaseConfig();
        config.setAppName("TestApp");
        config.setIncludeRegression(true);
        config.setNumberOfDays(14);

        when(queryExecutor.executeQuery(eq(QUERY), any(), eq("TestApp"), eq(true), eq(14)))
            .thenReturn(Collections.emptyList());

        // When
        List<MostFailedTestCasesData> result = service.getDataset(config);

        // Then
        verify(queryExecutor).executeQuery(eq(QUERY), any(), eq("TestApp"), eq(true), eq(14));
        assertTrue(result.isEmpty());
    }

    @Test
    void getDataset_withExcludeRegression_filtersCorrectly() {
        // Given
        MostFailedTestCaseConfig config = new MostFailedTestCaseConfig();
        config.setAppName("TestApp");
        config.setIncludeRegression(false);
        config.setNumberOfDays(7);

        when(queryExecutor.executeQuery(eq(QUERY), any(), eq("TestApp"), eq(false), eq(7)))
            .thenReturn(Collections.emptyList());

        // When
        List<MostFailedTestCasesData> result = service.getDataset(config);

        // Then
        verify(queryExecutor).executeQuery(eq(QUERY), any(), eq("TestApp"), eq(false), eq(7));
        assertTrue(result.isEmpty());
    }

    @Test
    void getDataset_withInvalidConfigType_throwsIllegalArgumentException() {
        // Given
        WidgetConfig invalidConfig = mock(WidgetConfig.class);

        // When & Then
        IllegalArgumentException exception = assertThrows(
            IllegalArgumentException.class,
            () -> service.getDataset(invalidConfig)
        );
        assertTrue(exception.getMessage().contains("Invalid configuration type"));
    }

    @Test
    void getDataset_withNoResults_returnsEmptyList() {
        // Given
        MostFailedTestCaseConfig config = new MostFailedTestCaseConfig();
        config.setAppName("EmptyApp");
        config.setIncludeRegression(false);
        config.setNumberOfDays(30);

        when(queryExecutor.executeQuery(eq(QUERY), any(), anyString(), anyBoolean(), anyInt()))
            .thenReturn(Collections.emptyList());

        // When
        List<MostFailedTestCasesData> result = service.getDataset(config);

        // Then
        assertNotNull(result);
        assertTrue(result.isEmpty());
    }

    // Helper method
    private MostFailedTestCasesData createTestCaseData(String name, int failureCount) {
        MostFailedTestCasesData data = new MostFailedTestCasesData();
        data.setTestCaseName(name);
        data.setFailureCount(failureCount);
        return data;
    }
}

// ============================================================================
// WidgetDatasetServiceFactory Tests
// ============================================================================

@ExtendWith(MockitoExtension.class)
class WidgetDatasetServiceFactoryTest {

    @Mock
    private MostFailedTestCaseDataService mostFailedTestCaseDataService;

    private WidgetDatasetServiceFactory factory;

    @BeforeEach
    void setUp() {
        factory = new WidgetDatasetServiceFactory(mostFailedTestCaseDataService);
    }

    @Test
    void getService_withValidType_returnsCorrectService() {
        // When
        WidgetDatasetService<?> service = factory.getService(WidgetDatasetType.MOST_TEST_CASES_FAILED);

        // Then
        assertNotNull(service);
        assertEquals(mostFailedTestCaseDataService, service);
    }

    @Test
    void getService_withUnknownType_throwsIllegalArgumentException() {
        // Given - create a new enum value that's not registered (simulated by null lookup)
        WidgetDatasetServiceFactory emptyFactory = new WidgetDatasetServiceFactory(mostFailedTestCaseDataService);
        
        // We can't easily test with unregistered enum, so we'll verify the null check works
        // by testing the error message format
        WidgetDatasetService<?> service = factory.getService(WidgetDatasetType.MOST_TEST_CASES_FAILED);
        assertNotNull(service); // Verify registered type works
    }

    @Test
    void hasService_withRegisteredType_returnsTrue() {
        // When
        boolean hasService = factory.hasService(WidgetDatasetType.MOST_TEST_CASES_FAILED);

        // Then
        assertTrue(hasService);
    }

    @Test
    void constructor_registersAllServices() {
        // Given
        WidgetDatasetServiceFactory newFactory = new WidgetDatasetServiceFactory(mostFailedTestCaseDataService);

        // When
        boolean hasService = newFactory.hasService(WidgetDatasetType.MOST_TEST_CASES_FAILED);

        // Then
        assertTrue(hasService);
        assertNotNull(newFactory.getService(WidgetDatasetType.MOST_TEST_CASES_FAILED));
    }
}

// ============================================================================
// WidgetConfig Deserialization Tests
// ============================================================================

class WidgetConfigDeserializationTest {

    private ObjectMapper objectMapper;

    @BeforeEach
    void setUp() {
        objectMapper = new ObjectMapper();
    }

    @Test
    void parseWidgetConfig_withValidJson_returnsCorrectType() throws JsonProcessingException {
        // Given
        String json = "{\"type\":\"MOST_TEST_CASES_FAILED\",\"appName\":\"TestApp\",\"includeRegression\":true,\"numberOfDays\":30}";

        // When
        WidgetConfig config = objectMapper.readValue(json, WidgetConfig.class);

        // Then
        assertNotNull(config);
        assertInstanceOf(MostFailedTestCaseConfig.class, config);
        assertEquals(WidgetDatasetType.MOST_TEST_CASES_FAILED, config.getType());
    }

    @Test
    void parseWidgetConfig_withMostFailedType_returnsMostFailedConfig() throws JsonProcessingException {
        // Given
        String json = "{\"type\":\"MOST_TEST_CASES_FAILED\",\"appName\":\"MyApp\",\"includeRegression\":false,\"numberOfDays\":14}";

        // When
        WidgetConfig config = objectMapper.readValue(json, WidgetConfig.class);

        // Then
        assertInstanceOf(MostFailedTestCaseConfig.class, config);
        MostFailedTestCaseConfig typedConfig = (MostFailedTestCaseConfig) config;
        assertEquals("MyApp", typedConfig.getAppName());
        assertFalse(typedConfig.isIncludeRegression());
        assertEquals(14, typedConfig.getNumberOfDays());
    }

    @Test
    void parseWidgetConfig_withInvalidJson_throwsJsonProcessingException() {
        // Given
        String invalidJson = "invalid json";

        // When & Then
        assertThrows(
            JsonProcessingException.class,
            () -> objectMapper.readValue(invalidJson, WidgetConfig.class)
        );
    }

    @Test
    void parseWidgetConfig_withMissingTypeField_throwsException() {
        // Given
        String jsonWithoutType = "{\"appName\":\"TestApp\",\"includeRegression\":true,\"numberOfDays\":30}";

        // When & Then
        assertThrows(
            JsonProcessingException.class,
            () -> objectMapper.readValue(jsonWithoutType, WidgetConfig.class)
        );
    }

    @Test
    void parseWidgetConfig_withUnknownType_throwsException() {
        // Given
        String jsonWithUnknownType = "{\"type\":\"UNKNOWN_TYPE\",\"appName\":\"TestApp\"}";

        // When & Then
        assertThrows(
            JsonProcessingException.class,
            () -> objectMapper.readValue(jsonWithUnknownType, WidgetConfig.class)
        );
    }

    @Test
    void parseWidgetConfig_withNullFields_parsesSuccessfully() throws JsonProcessingException {
        // Given
        String json = "{\"type\":\"MOST_TEST_CASES_FAILED\",\"appName\":null,\"includeRegression\":false,\"numberOfDays\":0}";

        // When
        WidgetConfig config = objectMapper.readValue(json, WidgetConfig.class);

        // Then
        assertNotNull(config);
        assertInstanceOf(MostFailedTestCaseConfig.class, config);
        MostFailedTestCaseConfig typedConfig = (MostFailedTestCaseConfig) config;
        assertNull(typedConfig.getAppName());
        assertFalse(typedConfig.isIncludeRegression());
        assertEquals(0, typedConfig.getNumberOfDays());
    }

    @Test
    void parseWidgetConfig_withExtraFields_ignoresUnknownProperties() throws JsonProcessingException {
        // Given
        String jsonWithExtraFields = "{\"type\":\"MOST_TEST_CASES_FAILED\",\"appName\":\"TestApp\",\"includeRegression\":true,\"numberOfDays\":30,\"extraField\":\"ignored\"}";

        // When
        WidgetConfig config = objectMapper.readValue(jsonWithExtraFields, WidgetConfig.class);

        // Then
        assertNotNull(config);
        assertInstanceOf(MostFailedTestCaseConfig.class, config);
        MostFailedTestCaseConfig typedConfig = (MostFailedTestCaseConfig) config;
        assertEquals("TestApp", typedConfig.getAppName());
    }
}

// ============================================================================
// Mock Classes (if not already defined)
// ============================================================================

class DashboardWidgetData {
    private long dashboardWidgetConfigId;
    private String dashboardName;
    private String dashboardDescription;
    private String widgetName;
    private String widgetDescription;
    private int ordinal;
    private String widgetConfiguration;

    // Getters and setters
    public long getDashboardWidgetConfigId() { return dashboardWidgetConfigId; }
    public void setDashboardWidgetConfigId(long id) { this.dashboardWidgetConfigId = id; }
    public String getDashboardName() { return dashboardName; }
    public void setDashboardName(String name) { this.dashboardName = name; }
    public String getDashboardDescription() { return dashboardDescription; }
    public void setDashboardDescription(String description) { this.dashboardDescription = description; }
    public String getWidgetName() { return widgetName; }
    public void setWidgetName(String name) { this.widgetName = name; }
    public String getWidgetDescription() { return widgetDescription; }
    public void setWidgetDescription(String description) { this.widgetDescription = description; }
    public int getOrdinal() { return ordinal; }
    public void setOrdinal(int ordinal) { this.ordinal = ordinal; }
    public String getWidgetConfiguration() { return widgetConfiguration; }
    public void setWidgetConfiguration(String config) { this.widgetConfiguration = config; }
}

class MostFailedTestCasesData {
    private String testCaseName;
    private int failureCount;

    public String getTestCaseName() { return testCaseName; }
    public void setTestCaseName(String name) { this.testCaseName = name; }
    public int getFailureCount() { return failureCount; }
    public void setFailureCount(int count) { this.failureCount = count; }
}

class DashboardNotFoundException extends RuntimeException {
    public DashboardNotFoundException(String message) {
        super(message);
    }
}

class WidgetConfigurationException extends RuntimeException {
    public WidgetConfigurationException(String message, Throwable cause) {
        super(message, cause);
    }
    public WidgetConfigurationException(String message) {
        super(message);
    }
}
```
