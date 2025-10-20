```java

// ==========================================
// 1. FieldMetadata Tests
// ==========================================
package com.yourpackage.dto;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class FieldMetadataTest {

    @Test
    void testNoArgsConstructor() {
        FieldMetadata metadata = new FieldMetadata();
        assertNull(metadata.getDisplayName());
        assertNull(metadata.getDataType());
        assertNull(metadata.getDescription());
    }

    @Test
    void testAllArgsConstructor() {
        FieldMetadata metadata = new FieldMetadata("App Name", "string", "Application name description");
        
        assertEquals("App Name", metadata.getDisplayName());
        assertEquals("string", metadata.getDataType());
        assertEquals("Application name description", metadata.getDescription());
    }

    @Test
    void testSettersAndGetters() {
        FieldMetadata metadata = new FieldMetadata();
        metadata.setDisplayName("Test Name");
        metadata.setDataType("integer");
        metadata.setDescription("Test description");
        
        assertEquals("Test Name", metadata.getDisplayName());
        assertEquals("integer", metadata.getDataType());
        assertEquals("Test description", metadata.getDescription());
    }

    @Test
    void testEqualsAndHashCode() {
        FieldMetadata metadata1 = new FieldMetadata("Name", "string", "Description");
        FieldMetadata metadata2 = new FieldMetadata("Name", "string", "Description");
        FieldMetadata metadata3 = new FieldMetadata("Other", "integer", "Other desc");
        
        assertEquals(metadata1, metadata2);
        assertNotEquals(metadata1, metadata3);
        assertEquals(metadata1.hashCode(), metadata2.hashCode());
    }
}

// ==========================================
// 2. WidgetConfigMetadata Tests
// ==========================================
package com.yourpackage.dto;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import java.util.HashMap;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class WidgetConfigMetadataTest {

    private WidgetConfigMetadata metadata;
    private Map<String, FieldMetadata> configFields;
    private Object exampleConfig;

    @BeforeEach
    void setUp() {
        metadata = new WidgetConfigMetadata();
        configFields = new HashMap<>();
        configFields.put("appName", new FieldMetadata("App Name", "string", "Application name"));
        exampleConfig = new Object();
    }

    @Test
    void testNoArgsConstructor() {
        WidgetConfigMetadata newMetadata = new WidgetConfigMetadata();
        assertNull(newMetadata.getConfigFields());
        assertNull(newMetadata.getExampleConfig());
    }

    @Test
    void testSetConfigFields() {
        metadata.setConfigFields(configFields);
        assertEquals(configFields, metadata.getConfigFields());
        assertEquals(1, metadata.getConfigFields().size());
    }

    @Test
    void testSetExampleConfig() {
        metadata.setExampleConfig(exampleConfig);
        assertEquals(exampleConfig, metadata.getExampleConfig());
    }

    @Test
    void testEmptyConfigFields() {
        metadata.setConfigFields(new HashMap<>());
        assertNotNull(metadata.getConfigFields());
        assertTrue(metadata.getConfigFields().isEmpty());
    }

    @Test
    void testNullConfigFields() {
        metadata.setConfigFields(null);
        assertNull(metadata.getConfigFields());
    }
}

// ==========================================
// 3. WidgetDataSetTypeDTO Tests
// ==========================================
package com.yourpackage.dto;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import java.util.HashMap;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class WidgetDataSetTypeDTOTest {

    private WidgetConfigMetadata widgetConfig;
    private Map<String, FieldMetadata> configFields;

    @BeforeEach
    void setUp() {
        configFields = new HashMap<>();
        configFields.put("appName", new FieldMetadata("App Name", "string", "Application name"));
        
        widgetConfig = new WidgetConfigMetadata();
        widgetConfig.setConfigFields(configFields);
    }

    @Test
    void testAllArgsConstructor() {
        WidgetDataSetTypeDTO dto = new WidgetDataSetTypeDTO(
            "MOST_TEST_CASES_FAILED",
            "Most Test Cases Failed",
            "BAR",
            widgetConfig
        );

        assertEquals("MOST_TEST_CASES_FAILED", dto.getType());
        assertEquals("Most Test Cases Failed", dto.getDisplayName());
        assertEquals("BAR", dto.getGraphType());
        assertNotNull(dto.getWidgetConfig());
        assertEquals(widgetConfig, dto.getWidgetConfig());
    }

    @Test
    void testNoArgsConstructor() {
        WidgetDataSetTypeDTO dto = new WidgetDataSetTypeDTO();
        assertNull(dto.getType());
        assertNull(dto.getDisplayName());
        assertNull(dto.getGraphType());
        assertNull(dto.getWidgetConfig());
    }

    @Test
    void testSettersAndGetters() {
        WidgetDataSetTypeDTO dto = new WidgetDataSetTypeDTO();
        dto.setType("TEST_TYPE");
        dto.setDisplayName("Test Display Name");
        dto.setGraphType("LINE");
        dto.setWidgetConfig(widgetConfig);

        assertEquals("TEST_TYPE", dto.getType());
        assertEquals("Test Display Name", dto.getDisplayName());
        assertEquals("LINE", dto.getGraphType());
        assertEquals(widgetConfig, dto.getWidgetConfig());
    }
}

// ==========================================
// 4. ConfigMetadataExtractor Tests
// ==========================================
package com.yourpackage.util;

import com.yourpackage.config.WidgetConfig;
import com.yourpackage.dto.FieldMetadata;
import com.yourpackage.annotation.ConfigField;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class ConfigMetadataExtractorTest {

    // Test config class
    static class TestConfig implements WidgetConfig {
        @ConfigField(displayName = "Test String", description = "A test string field")
        private String stringField;

        @ConfigField(displayName = "Test Int", description = "A test integer field")
        private int intField;

        @ConfigField(displayName = "Test Boolean", description = "A test boolean field")
        private boolean booleanField;

        @ConfigField(displayName = "Test Long", description = "A test long field")
        private long longField;

        @ConfigField(displayName = "Test Double", description = "A test double field")
        private double doubleField;

        // Field without annotation - should be ignored
        private String ignoredField;
    }

    static class EmptyConfig implements WidgetConfig {
        private String noAnnotation;
    }

    @Test
    void testExtractMetadataWithAllFieldTypes() {
        Map<String, FieldMetadata> metadata = ConfigMetadataExtractor.extractMetadata(TestConfig.class);

        assertEquals(5, metadata.size());
        
        // Test string field
        assertTrue(metadata.containsKey("stringField"));
        FieldMetadata stringMeta = metadata.get("stringField");
        assertEquals("Test String", stringMeta.getDisplayName());
        assertEquals("string", stringMeta.getDataType());
        assertEquals("A test string field", stringMeta.getDescription());

        // Test int field
        assertTrue(metadata.containsKey("intField"));
        FieldMetadata intMeta = metadata.get("intField");
        assertEquals("Test Int", intMeta.getDisplayName());
        assertEquals("integer", intMeta.getDataType());

        // Test boolean field
        assertTrue(metadata.containsKey("booleanField"));
        assertEquals("boolean", metadata.get("booleanField").getDataType());

        // Test long field
        assertTrue(metadata.containsKey("longField"));
        assertEquals("long", metadata.get("longField").getDataType());

        // Test double field
        assertTrue(metadata.containsKey("doubleField"));
        assertEquals("double", metadata.get("doubleField").getDataType());

        // Ignored field should not be present
        assertFalse(metadata.containsKey("ignoredField"));
    }

    @Test
    void testExtractMetadataWithNoAnnotations() {
        Map<String, FieldMetadata> metadata = ConfigMetadataExtractor.extractMetadata(EmptyConfig.class);
        assertTrue(metadata.isEmpty());
    }

    @Test
    void testExtractMetadataWithCustomDataType() {
        class CustomConfig implements WidgetConfig {
            @ConfigField(displayName = "Custom", description = "Custom type", dataType = "custom-type")
            private String field;
        }

        Map<String, FieldMetadata> metadata = ConfigMetadataExtractor.extractMetadata(CustomConfig.class);
        assertEquals("custom-type", metadata.get("field").getDataType());
    }

    @Test
    void testExtractMetadataPreservesInsertionOrder() {
        Map<String, FieldMetadata> metadata = ConfigMetadataExtractor.extractMetadata(TestConfig.class);
        
        // LinkedHashMap should preserve field order
        String[] keys = metadata.keySet().toArray(new String[0]);
        assertEquals("stringField", keys[0]);
        assertEquals("intField", keys[1]);
        assertEquals("booleanField", keys[2]);
    }
}

// ==========================================
// 5. MostFailedTestCaseConfig Tests
// ==========================================
package com.yourpackage.config;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class MostFailedTestCaseConfigTest {

    @Test
    void testDefaultConfig() {
        MostFailedTestCaseConfig config = MostFailedTestCaseConfig.defaultConfig();

        assertNotNull(config);
        assertEquals("Fusion Algo", config.getAppName());
        assertEquals(14, config.getDays());
        assertTrue(config.isIncludeRegression());
    }

    @Test
    void testSettersAndGetters() {
        MostFailedTestCaseConfig config = new MostFailedTestCaseConfig();
        
        config.setAppName("Test App");
        config.setDays(30);
        config.setIncludeRegression(false);

        assertEquals("Test App", config.getAppName());
        assertEquals(30, config.getDays());
        assertFalse(config.isIncludeRegression());
    }

    @Test
    void testConfigFieldAnnotations() throws NoSuchFieldException {
        // Verify annotations are present
        assertTrue(MostFailedTestCaseConfig.class
            .getDeclaredField("appName")
            .isAnnotationPresent(ConfigField.class));
        
        assertTrue(MostFailedTestCaseConfig.class
            .getDeclaredField("days")
            .isAnnotationPresent(ConfigField.class));
        
        assertTrue(MostFailedTestCaseConfig.class
            .getDeclaredField("includeRegression")
            .isAnnotationPresent(ConfigField.class));
    }

    @Test
    void testAppNameFieldAnnotationValues() throws NoSuchFieldException {
        ConfigField annotation = MostFailedTestCaseConfig.class
            .getDeclaredField("appName")
            .getAnnotation(ConfigField.class);

        assertEquals("App Name", annotation.displayName());
        assertEquals("The name of the application to filter test results for", annotation.description());
    }

    @Test
    void testBoundaryValues() {
        MostFailedTestCaseConfig config = new MostFailedTestCaseConfig();
        
        // Test with 0 days
        config.setDays(0);
        assertEquals(0, config.getDays());
        
        // Test with large number
        config.setDays(365);
        assertEquals(365, config.getDays());
        
        // Test with null app name
        config.setAppName(null);
        assertNull(config.getAppName());
        
        // Test with empty app name
        config.setAppName("");
        assertEquals("", config.getAppName());
    }
}

// ==========================================
// 6. WidgetDatasetType Enum Tests
// ==========================================
package com.yourpackage.enums;

import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class WidgetDatasetTypeTest {

    @Test
    void testEnumValues() {
        WidgetDatasetType[] values = WidgetDatasetType.values();
        assertTrue(values.length > 0);
        assertNotNull(WidgetDatasetType.valueOf("MOST_TEST_CASES_FAILED"));
    }

    @Test
    void testGetDisplayName() {
        String displayName = WidgetDatasetType.MOST_TEST_CASES_FAILED.getDisplayName();
        assertEquals("Most Test Cases Failed", displayName);
    }

    @Test
    void testGetGraphType() {
        String graphType = WidgetDatasetType.MOST_TEST_CASES_FAILED.getGraphType();
        assertEquals("BAR", graphType);
    }

    @Test
    void testGetDefaultConfig() {
        Object config = WidgetDatasetType.MOST_TEST_CASES_FAILED.getDefaultConfig();
        assertNotNull(config);
        assertTrue(config instanceof MostFailedTestCaseConfig);
        
        MostFailedTestCaseConfig typedConfig = (MostFailedTestCaseConfig) config;
        assertEquals("Fusion Algo", typedConfig.getAppName());
        assertEquals(14, typedConfig.getDays());
        assertTrue(typedConfig.isIncludeRegression());
    }

    @Test
    void testGetFieldMetadata() {
        Map<String, FieldMetadata> metadata = WidgetDatasetType.MOST_TEST_CASES_FAILED.getFieldMetadata();
        
        assertNotNull(metadata);
        assertEquals(3, metadata.size());
        assertTrue(metadata.containsKey("appName"));
        assertTrue(metadata.containsKey("days"));
        assertTrue(metadata.containsKey("includeRegression"));
        
        // Verify field metadata details
        FieldMetadata appNameMeta = metadata.get("appName");
        assertEquals("App Name", appNameMeta.getDisplayName());
        assertEquals("string", appNameMeta.getDataType());
    }

    @Test
    void testEnumName() {
        assertEquals("MOST_TEST_CASES_FAILED", WidgetDatasetType.MOST_TEST_CASES_FAILED.name());
    }
}

// ==========================================
// 7. Controller Tests
// ==========================================
package com.yourpackage.controller;

import com.yourpackage.dto.WidgetDataSetTypeDTO;
import com.yourpackage.enums.WidgetDatasetType;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

@ExtendWith(MockitoExtension.class)
class WidgetControllerTest {

    @InjectMocks
    private WidgetController widgetController;  // Your controller class name

    @Test
    void testGetAvailableWidgetConfigs() {
        ResponseEntity<List<WidgetDataSetTypeDTO>> response = widgetController.getAvailableWidgetConfigs();

        assertNotNull(response);
        assertEquals(HttpStatus.OK, response.getStatusCode());
        assertNotNull(response.getBody());
        
        List<WidgetDataSetTypeDTO> body = response.getBody();
        assertFalse(body.isEmpty());
        assertEquals(WidgetDatasetType.values().length, body.size());
    }

    @Test
    void testGetAvailableWidgetConfigsReturnsCorrectStructure() {
        ResponseEntity<List<WidgetDataSetTypeDTO>> response = widgetController.getAvailableWidgetConfigs();
        List<WidgetDataSetTypeDTO> body = response.getBody();

        WidgetDataSetTypeDTO dto = body.get(0);
        assertNotNull(dto.getType());
        assertNotNull(dto.getDisplayName());
        assertNotNull(dto.getGraphType());
        assertNotNull(dto.getWidgetConfig());
        assertNotNull(dto.getWidgetConfig().getConfigFields());
        assertNotNull(dto.getWidgetConfig().getExampleConfig());
    }

    @Test
    void testGetAvailableWidgetConfigsContainsMostTestCasesFailed() {
        ResponseEntity<List<WidgetDataSetTypeDTO>> response = widgetController.getAvailableWidgetConfigs();
        List<WidgetDataSetTypeDTO> body = response.getBody();

        boolean found = body.stream()
            .anyMatch(dto -> "MOST_TEST_CASES_FAILED".equals(dto.getType()));
        
        assertTrue(found, "Should contain MOST_TEST_CASES_FAILED widget");
    }

    @Test
    void testGetAvailableWidgetConfigsFieldMetadata() {
        ResponseEntity<List<WidgetDataSetTypeDTO>> response = widgetController.getAvailableWidgetConfigs();
        List<WidgetDataSetTypeDTO> body = response.getBody();

        WidgetDataSetTypeDTO mostFailedDto = body.stream()
            .filter(dto -> "MOST_TEST_CASES_FAILED".equals(dto.getType()))
            .findFirst()
            .orElseThrow();

        assertEquals("Most Test Cases Failed", mostFailedDto.getDisplayName());
        assertEquals("BAR", mostFailedDto.getGraphType());
        
        var configFields = mostFailedDto.getWidgetConfig().getConfigFields();
        assertEquals(3, configFields.size());
        assertTrue(configFields.containsKey("appName"));
        assertTrue(configFields.containsKey("days"));
        assertTrue(configFields.containsKey("includeRegression"));
    }
}

```
