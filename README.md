```java

private WidgetResponse mapToWidgetResponse(DashboardWidgetData data) {
    try {
        // 1. Parse the widget configuration JSON into a WidgetConfig object
        var widgetConfig = JsonUtil.getObjectMapper()
                .readValue(data.getWidgetConfiguration(), WidgetConfig.class);

        // 2. Parse again to extract the dataset type (e.g. "MOST_FAILED_TEST_CASES")
        var jsonNode = JsonUtil.getObjectMapper()
                .readTree(data.getWidgetConfiguration());
        var typeValue = jsonNode.get("type").asText();
        var datasetType = WidgetDatasetType.valueOf(typeValue);

        // 3. Resolve the correct dataset service based on the type
        WidgetDatasetService<?> service = factory.getService(datasetType);

        // 4. Execute the dataset query using the widget config
        List<?> dataset = service.getDataset(widgetConfig);

        // 5. Build the WidgetResponse
        var widgetResponse = new WidgetResponse();
        widgetResponse.setWidgetId(data.getDashboardWidgetConfigId());
        widgetResponse.setOrdinal(data.getOrdinal());
        widgetResponse.setName(data.getWidgetName());
        widgetResponse.setDescription(data.getWidgetDescription());
        widgetResponse.setWidgetDataSetType(datasetType);
        widgetResponse.setWidgetConfig(widgetConfig);
        widgetResponse.setDataSets((List<Object>) dataset); // safe cast under our control

        return widgetResponse;

    } catch (JsonProcessingException e) {
        throw new WidgetConfigurationException(
                "Failed to parse widget configuration for widget: " + data.getWidgetName(), e
        );
    }
}

```
