```java

/**
 * Configuration object for the {@link WidgetDatasetType#MOST_TEST_CASES_FAILED}
 * dataset. This config determines which application, how many days to look back,
 * and whether regression test cases should be included when generating the widget
 * dataset.
 * <p>
 * This class is deserialized polymorphically using the widget {@code type} field
 * via Jackson annotations defined on {@link WidgetConfig}.
 * </p>
 */
@EqualsAndHashCode(callSuper = true)
@Data
@JsonTypeName("MOST_TEST_CASES_FAILED")
@JsonIgnoreProperties(ignoreUnknown = true)
public final class MostFailedTestCaseConfig extends WidgetConfig {

    /**
     * Name of the application to filter test cases for.
     * <p>
     * This value is optional in the database, and defaults to {@code "Fusion Algo"}.
     * </p>
     */
    @ConfigField(
            displayName = "Application Name",
            description = "Name of the application to filter test cases",
            defaultValue = "Fusion Algo"
    )
    private String appName;

    /**
     * Number of days to look back when evaluating failed test cases.
     * <p>
     * Must be greater than zero. Defaults to {@code 14}.
     * </p>
     */
    @ConfigField(
            displayName = "Number of Days",
            description = "Number of days to look back for failed test cases",
            defaultValue = "14"
    )
    private int numberOfDays;

    /**
     * Whether regression test cases should be included in the results.
     * <p>
     * Defaults to {@code true}.
     * </p>
     */
    @ConfigField(
            displayName = "Include Regression",
            description = "Include regression test cases in the results",
            defaultValue = "true"
    )
    private boolean includeRegression;

    /**
     * Creates the default configuration for this widget type.
     * <p>
     * Used as a fallback for any missing or invalid values in the database-configured version.
     * </p>
     *
     * @return a fully populated {@link MostFailedTestCaseConfig} instance with default values
     */
    public static MostFailedTestCaseConfig defaultConfig() {
        MostFailedTestCaseConfig config = new MostFailedTestCaseConfig();
        config.setAppName("Fusion Algo");
        config.setNumberOfDays(14);
        config.setIncludeRegression(true);
        return config;
    }

    /**
     * Returns the widget dataset type associated with this config.
     *
     * @return {@link WidgetDatasetType#MOST_TEST_CASES_FAILED}
     */
    @Override
    public WidgetDatasetType getType() {
        return WidgetDatasetType.MOST_TEST_CASES_FAILED;
    }

    /**
     * Converts this configuration into a simple map of field names to values.
     * <p>
     * This allows the dashboard service to introspect config values <b>without using reflection</b>
     * when producing {@link FieldMetadata}.
     * </p>
     *
     * @return a map containing {@code appName}, {@code numberOfDays}, and {@code includeRegression}
     */
    @Override
    public Map<String, Object> toValueMap() {
        Map<String, Object> map = new LinkedHashMap<>();
        map.put("appName", appName);
        map.put("numberOfDays", numberOfDays);
        map.put("includeRegression", includeRegression);
        return map;
    }

    /**
     * Merges the current configuration with the provided default configuration.
     * <p>
     * Rules for merging:
     * <ul>
     *     <li><b>Strings:</b> non-null and non-blank values override defaults</li>
     *     <li><b>Integers:</b> must be &gt; 0 to override defaults</li>
     *     <li><b>Booleans:</b> always taken as-is, since all boolean values are valid</li>
     * </ul>
     * </p>
     *
     * @param defaults the default config to fall back to
     * @return a new {@link MostFailedTestCaseConfig} instance after merging
     */
    @Override
    public WidgetConfig mergeWith(WidgetConfig defaults) {
        MostFailedTestCaseConfig d = (MostFailedTestCaseConfig) defaults;
        MostFailedTestCaseConfig merged = new MostFailedTestCaseConfig();

        merged.setAppName(
                this.appName != null && !this.appName.isBlank()
                        ? this.appName
                        : d.appName
        );

        merged.setNumberOfDays(
                this.numberOfDays > 0
                        ? this.numberOfDays
                        : d.numberOfDays
        );

        merged.setIncludeRegression(this.includeRegression);

        return merged;
    }
}


```
