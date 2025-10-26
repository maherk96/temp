```java

private WidgetResponse mapToWidgetResponse(DashboardWidgetData data) {
    try {
        var widgetConfig = JsonUtil.getObjectMapper()
            .readValue(data.getWidgetConfiguration(), WidgetConfig.class);

        var jsonNode = JsonUtil.getObjectMapper()
            .readTree(data.getWidgetConfiguration());
        var typeValue = jsonNode.get("type").asText();
        var datasetType = WidgetDatasetType.valueOf(typeValue);

        var service = factory.getService(datasetType);
        var dataset = service.getDataset(widgetConfig);

        var widgetResponse = new WidgetResponse();
        widgetResponse.setWidgetId(data.getDashboardWidgetConfigId());
        widgetResponse.setOrdinal(data.getOrdinal());
        widgetResponse.setName(data.getWidgetName());
        widgetResponse.setDescription(data.getWidgetDescription());
        widgetResponse.setDatasetType(datasetType);
        widgetResponse.setWidgetConfig(widgetConfig);
        widgetResponse.setDataset(dataset);

        return widgetResponse;

    } catch (JsonProcessingException e) {
        throw new WidgetConfigurationException(
            "Failed to parse widget configuration for widget: " + data.getWidgetName(), e
        );
    }
}

```
