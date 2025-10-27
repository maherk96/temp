```java

package com.example.dashboard;

import com.fasterxml.jackson.annotation.JsonSubTypes;
import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.core.JsonProcessingException;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.EnumMap;
import java.util.List;
import java.util.Map;

// ============================================================================
// DTOs
// ============================================================================

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
class DashboardDatasetResponse {
    private long id;
    private String name;
    private String description;
    private List<WidgetResponse> widgetResponseList;
}

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
class WidgetResponse {
    private long widgetId;
    private int ordinal;
    private WidgetDatasetType widgetDataSetType;
    private String graphType;
    private String name;
    private String description;
    private WidgetConfig widgetConfig;
    private List<Object> datasets;
}

// ============================================================================
// Models
// ============================================================================

@JsonTypeInfo(
    use = JsonTypeInfo.Id.NAME,
    include = JsonTypeInfo.As.PROPERTY,
    property = "type",
    visible = true
)
@JsonSubTypes({
    @JsonSubTypes.Type(value = MostFailedTestCaseConfig.class, name = "MOST_TEST_CASES_FAILED")
})
abstract class WidgetConfig {
    public abstract WidgetDatasetType getType();
}

@Data
class MostFailedTestCaseConfig extends WidgetConfig {
    private String appName;
    private boolean includeRegression;
    private int numberOfDays;
    
    @Override
    public WidgetDatasetType getType() {
        return WidgetDatasetType.MOST_TEST_CASES_FAILED;
    }
}

enum WidgetDatasetType {
    MOST_TEST_CASES_FAILED
}

// ============================================================================
// Services
// ============================================================================

@Slf4j
@Service
class DashboardDataService {

    private static final String DASHBOARD_QUERY_KEY = "DashboardDatasetQuery";

    private final DatabaseQueryExecutor queryExecutor;
    private final Map<String, String> qapQueries;
    private final WidgetDatasetServiceFactory widgetServiceFactory;

    @Autowired
    public DashboardDataService(DatabaseQueryExecutor queryExecutor,
                                Map<String, String> qapQueries,
                                WidgetDatasetServiceFactory widgetServiceFactory) {
        this.queryExecutor = queryExecutor;
        this.qapQueries = qapQueries;
        this.widgetServiceFactory = widgetServiceFactory;
    }

    public DashboardDatasetResponse getDashboardData(long dashboardId) {
        log.debug("Retrieving dashboard data for dashboardId: {}", dashboardId);
        
        List<DashboardWidgetData> data = queryExecutor.executeQuery(
            qapQueries.get(DASHBOARD_QUERY_KEY),
            new DashboardWidgetDataMapper(),
            dashboardId
        );

        if (data.isEmpty()) {
            log.warn("Dashboard not found: {}", dashboardId);
            throw new DashboardNotFoundException("Dashboard not found: " + dashboardId);
        }

        DashboardWidgetData firstRow = data.get(0);
        DashboardDatasetResponse response = buildDashboardResponse(dashboardId, firstRow);
        
        List<WidgetResponse> widgets = data.stream()
            .map(this::mapToWidgetResponse)
            .toList();

        response.setWidgetResponseList(widgets);
        
        log.info("Successfully retrieved dashboard data for dashboardId: {} with {} widgets", 
                 dashboardId, widgets.size());
        
        return response;
    }

    private DashboardDatasetResponse buildDashboardResponse(long dashboardId, DashboardWidgetData data) {
        return DashboardDatasetResponse.builder()
            .id(dashboardId)
            .name(data.getDashboardName())
            .description(data.getDashboardDescription())
            .build();
    }

    private WidgetResponse mapToWidgetResponse(DashboardWidgetData data) {
        try {
            log.debug("Mapping widget: {} (ID: {})", data.getWidgetName(), data.getDashboardWidgetConfigId());
            
            WidgetConfig widgetConfig = parseWidgetConfig(data.getWidgetConfiguration());
            WidgetDatasetType datasetType = widgetConfig.getType();
            
            WidgetDatasetService<?> service = widgetServiceFactory.getService(datasetType);
            List<?> dataset = service.getDataset(widgetConfig);

            return buildWidgetResponse(data, datasetType, widgetConfig, dataset);
            
        } catch (JsonProcessingException e) {
            log.error("Failed to parse widget configuration for widget: {}", data.getWidgetName(), e);
            throw new WidgetConfigurationException(
                "Failed to parse widget configuration for widget: " + data.getWidgetName(), e
            );
        } catch (IllegalArgumentException e) {
            log.error("Invalid widget type for widget: {}", data.getWidgetName(), e);
            throw new WidgetConfigurationException(
                "Invalid widget type for widget: " + data.getWidgetName(), e
            );
        }
    }

    private WidgetConfig parseWidgetConfig(String configJson) throws JsonProcessingException {
        return JsonUtil.getObjectMapper().readValue(configJson, WidgetConfig.class);
    }

    private WidgetResponse buildWidgetResponse(DashboardWidgetData data,
                                                WidgetDatasetType datasetType,
                                                WidgetConfig widgetConfig,
                                                List<?> dataset) {
        log.debug("Built widget response for widget: {} with {} data points", 
                  data.getWidgetName(), dataset.size());
        
        return WidgetResponse.builder()
            .widgetId(data.getDashboardWidgetConfigId())
            .ordinal(data.getOrdinal())
            .name(data.getWidgetName())
            .description(data.getWidgetDescription())
            .widgetDataSetType(datasetType)
            .widgetConfig(widgetConfig)
            .datasets(new ArrayList<>(dataset))
            .build();
    }
}

// ============================================================================
// Widget Dataset Services
// ============================================================================

interface WidgetDatasetService<T> {
    List<T> getDataset(WidgetConfig widgetConfig);
}

@Slf4j
@Service
class MostFailedTestCaseDataService implements WidgetDatasetService<MostFailedTestCasesData> {

    private static final String QUERY_KEY = "MostFailedTestCasesQuery";

    private final DatabaseQueryExecutor queryExecutor;
    private final Map<String, String> qapQueries;

    @Autowired
    public MostFailedTestCaseDataService(DatabaseQueryExecutor queryExecutor, 
                                         Map<String, String> qapQueries) {
        this.queryExecutor = queryExecutor;
        this.qapQueries = qapQueries;
    }

    @Override
    public List<MostFailedTestCasesData> getDataset(WidgetConfig widgetConfig) {
        if (!(widgetConfig instanceof MostFailedTestCaseConfig)) {
            log.error("Invalid config type provided: {}. Expected: MostFailedTestCaseConfig", 
                      widgetConfig.getClass().getName());
            throw new IllegalArgumentException(
                "Invalid configuration type. Expected MostFailedTestCaseConfig but received: " 
                + widgetConfig.getClass().getSimpleName()
            );
        }

        MostFailedTestCaseConfig config = (MostFailedTestCaseConfig) widgetConfig;
        
        log.debug("Retrieving most failed test cases for app: {}, includeRegression: {}, days: {}", 
                  config.getAppName(), config.isIncludeRegression(), config.getNumberOfDays());

        List<MostFailedTestCasesData> results = queryExecutor.executeQuery(
            qapQueries.get(QUERY_KEY),
            new MostFailedTestCasesDataMapper(),
            config.getAppName(),
            config.isIncludeRegression(),
            config.getNumberOfDays()
        );
        
        log.info("Retrieved {} failed test cases for app: {}", results.size(), config.getAppName());
        
        return results;
    }
}

// ============================================================================
// Factory
// ============================================================================

@Slf4j
@Service
class WidgetDatasetServiceFactory {

    private final Map<WidgetDatasetType, WidgetDatasetService<?>> services;

    @Autowired
    public WidgetDatasetServiceFactory(MostFailedTestCaseDataService mostFailedTestCaseDataService) {
        this.services = new EnumMap<>(WidgetDatasetType.class);
        services.put(WidgetDatasetType.MOST_TEST_CASES_FAILED, mostFailedTestCaseDataService);
        
        log.info("Initialized WidgetDatasetServiceFactory with {} widget type(s)", services.size());
        logRegisteredServices();
    }

    public WidgetDatasetService<?> getService(WidgetDatasetType type) {
        log.debug("Retrieving service for widget type: {}", type);
        
        WidgetDatasetService<?> service = services.get(type);
        
        if (service == null) {
            log.error("No service registered for widget type: {}", type);
            throw new IllegalArgumentException(
                String.format("No service found for widget type: %s. Available types: %s", 
                             type, services.keySet())
            );
        }
        
        return service;
    }

    public boolean hasService(WidgetDatasetType type) {
        return services.containsKey(type);
    }

    private void logRegisteredServices() {
        if (log.isDebugEnabled()) {
            services.forEach((type, service) -> 
                log.debug("Registered service: {} -> {}", type, service.getClass().getSimpleName())
            );
        }
    }
}
```
