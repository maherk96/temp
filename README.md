```java
@Slf4j
@Service
public class DashboardDataService {

    private static final String DASHBOARD_QUERY_KEY = "DashboardDatasetQuery";

    private final DatabaseQueryExecutor queryExecutor;
    private final Map<String, String> qapQueries;
    private final WidgetDatasetServiceFactory widgetServiceFactory;

    @Autowired
    public DashboardDataService(
            DatabaseQueryExecutor queryExecutor,
            Map<String, String> qapQueries,
            WidgetDatasetServiceFactory widgetServiceFactory) {

        this.queryExecutor = queryExecutor;
        this.qapQueries = qapQueries;
        this.widgetServiceFactory = widgetServiceFactory;
    }

    /**
     * Retrieves all widget data belonging to a dashboard.
     *
     * @param dashboardId the ID of the dashboard
     * @return populated {@link DashboardDatasetResponse}
     */
    public DashboardDatasetResponse getDashboardData(long dashboardId) {
        log.debug("Retrieving dashboard data for dashboardId: {}", dashboardId);

        List<DashboardWidgetData> rows =
                queryExecutor.executeQuery(
                        qapQueries.get(DASHBOARD_QUERY_KEY),
                        new DashboardWidgetDataMapper(),
                        dashboardId);

        if (rows.isEmpty()) {
            log.warn("Dashboard not found: {}", dashboardId);
            throw new DashboardNotFoundException("Dashboard not found: " + dashboardId);
        }

        DashboardWidgetData firstRow = rows.get(0);
        DashboardDatasetResponse response =
                buildDashboardResponse(dashboardId, firstRow);

        List<WidgetResponse> widgets =
                rows.stream()
                    .map(this::mapToWidgetResponse)
                    .toList();

        response.setWidgetResponseList(widgets);

        log.info(
                "Successfully retrieved dashboard data for dashboardId: {} with {} widgets",
                dashboardId,
                widgets.size());

        return response;
    }

    /**
     * Builds the top-level dashboard metadata (name, description, etc).
     */
    private DashboardDatasetResponse buildDashboardResponse(
            long dashboardId, DashboardWidgetData data) {

        return DashboardDatasetResponse.builder()
                .id(dashboardId)
                .name(data.getDashboardName())
                .description(data.getDashboardDescription())
                .build();
    }

    /**
     * Converts one widget DB row into a fully populated {@link WidgetResponse}.
     *
     * @param data row retrieved from the database
     * @return built widget response with metadata + dataset values
     */
    private WidgetResponse mapToWidgetResponse(DashboardWidgetData data) {
        log.debug("Mapping widget: {} (ID: {}), configId: {}",
                data.getWidgetName(),
                data.getDashboardWidgetConfigId(),
                data.getWidgetConfigId());

        try {
            WidgetConfig widgetConfig = parseAndMergeConfig(data.getWidgetConfiguration());
            WidgetDatasetType type = widgetConfig.getType();

            WidgetDatasetService<?> service = widgetServiceFactory.getService(type);
            List<?> dataset = service.getDataset(widgetConfig);

            return buildWidgetResponse(data, type, widgetConfig, dataset);

        } catch (JsonProcessingException e) {
            log.error("Failed to parse widget configuration for widget: {}", data.getWidgetName(), e);
            throw new WidgetConfigurationException(
                    "Failed to parse widget configuration for widget: " + data.getWidgetName(), e);

        } catch (IllegalArgumentException e) {
            log.error("Invalid widget type for widget: {}", data.getWidgetName(), e);
            throw new WidgetConfigurationException(
                    "Invalid widget type for widget: " + data.getWidgetName(), e);
        }
    }

    /**
     * Parses raw JSON config and merges missing values with the widget type's default config.
     */
    private WidgetConfig parseAndMergeConfig(String json) throws JsonProcessingException {
        WidgetConfig parsed = JsoUnitil.getObjectMapper().readValue(json, WidgetConfig.class);
        WidgetConfig defaults = parsed.getType().getDefaultConfig();
        return parsed.mergeWith(defaults);
    }

    /**
     * Builds the final unified WidgetResponse object.
     */
    private WidgetResponse buildWidgetResponse(
            DashboardWidgetData data,
            WidgetDatasetType datasetType,
            WidgetConfig widgetConfig,
            List<?> dataset) {

        log.debug("Built widget response for widget: {} with {} data points",
                data.getWidgetName(),
                dataset.size());

        Map<String, FieldMetadata> metadata =
                createConfigMetadataWithValues(datasetType, widgetConfig);

        return WidgetResponse.builder()
                .widgetId(data.getDashboardWidgetConfigId())
                .ordinal(data.getOrdinal())
                .name(data.getWidgetName())
                .widgetDataSetType(datasetType)
                .graphType(data.getGraphType())
                .description(data.getWidgetDescription())
                .widgetConfig(metadata)
                .dataSets(new ArrayList<>(dataset))
                .build();
    }

    /**
     * Produces a map of field metadata for each config property,
     * enriched with the actual config values from {@link WidgetConfig#toValueMap()}.
     */
    private Map<String, FieldMetadata> createConfigMetadataWithValues(
            WidgetDatasetType datasetType,
            WidgetConfig config) {

        Map<String, FieldMetadata> baseMeta = datasetType.getFieldMetadata();
        Map<String, Object> values = config.toValueMap();

        Map<String, FieldMetadata> result = new LinkedHashMap<>();

        baseMeta.forEach((fieldName, meta) -> {
            Object value = values.get(fieldName);

            result.put(fieldName, new FieldMetadata(
                    meta.getDisplayName(),
                    meta.getDataType(),
                    meta.getDescription(),
                    value
            ));
        });

        return result;
    }
}
```
