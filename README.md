# Refactoring Prompt: DashboardDataService

## Context
You are refactoring a Spring service class that retrieves dashboard data with widgets. The class currently has issues with reflection usage, poor separation of concerns, and fragile error handling.

## Requirements

### 1. Fix Critical Issues
- **Typo**: Change `JsoUnitil` to `JsonUtil` in `parseWidgetConfig`
- **Remove Reflection**: Eliminate all reflection-based field access in `getFieldValue`, `isValidValue`, and related methods
- **Null Safety**: Add proper null checks and use `Optional` where appropriate

### 2. Improve Architecture

#### Separate Concerns into Multiple Classes:
```
DashboardDataService (orchestration only)
├── WidgetMapper (widget mapping logic)
├── ConfigMetadataBuilder (metadata creation)
└── WidgetConfigParser (configuration parsing)
```

#### Alternative Approach for Config Values:
Replace reflection with one of these approaches:
- Option A: Make `WidgetConfig` implement a method like `Map<String, Object> toConfigMap()`
- Option B: Use Jackson's `@JsonAnyGetter` to capture all fields as a map
- Option C: Define explicit getter methods for each config field

### 3. Enhance Error Handling

Create specific exception types:
- `WidgetConfigParsingException` - for JSON parsing errors
- `InvalidWidgetTypeException` - for invalid widget types
- `WidgetProcessingException` - for widget processing errors

Implement graceful degradation:
- If one widget fails, log the error and continue processing other widgets
- Return partial dashboard data with error indicators for failed widgets
- Add a `List<WidgetError>` field to `DashboardDatasetResponse`

### 4. Code Quality Improvements

**Stream Processing:**
```java
// Replace error-prone mapping with safe version
private Optional<WidgetResponse> mapToWidgetResponseSafely(DashboardWidgetData data) {
    try {
        return Optional.of(mapToWidgetResponse(data));
    } catch (Exception e) {
        log.error("Failed to map widget: {}", data.getWidgetName(), e);
        return Optional.empty();
    }
}
```

**Type Safety:**
- Add proper generics to `WidgetDatasetService<T>` instead of using `<?>`
- Avoid raw types and `List<?>`

**Configuration:**
- Either implement `mergeWithDefaults` properly or remove it
- Simplify default value logic (consider using Java's `ObjectUtils.defaultIfNull`)

### 5. Additional Enhancements

**Logging:**
- Reduce redundant log statements
- Use structured logging with meaningful context
- Remove debug logs that don't add value

**Performance:**
- Consider adding `@Cacheable` annotation for dashboard data if appropriate
- Evaluate if `LinkedHashMap` instantiation in `createConfigMetadataWithValues` can be optimized

**Validation:**
- Add validation for `dashboardId` parameter
- Validate widget configuration before processing

### 6. Testing Considerations

Ensure the refactored code is testable:
- Make methods package-private or protected for testing
- Avoid static methods where possible
- Inject all dependencies
- Consider using constructor injection only (already done, good!)

## Output Format

Provide:
1. **Refactored `DashboardDataService`** - slimmed down orchestration class
2. **New supporting classes** - `WidgetMapper`, `ConfigMetadataBuilder`, etc.
3. **Custom exception classes** - with appropriate hierarchy
4. **Key changes summary** - bullet list of what was improved and why

## Constraints

- Maintain backward compatibility with existing public API (`getDashboardData` method signature)
- Keep all existing functionality working
- Use Spring Framework best practices
- Follow SOLID principles, especially Single Responsibility
- Maintain the existing dependency injection structure
- Keep Lombok annotations where they add value

## Code Style

- Use Java 17+ features where appropriate
- Follow standard Java naming conventions
- Add JavaDoc for public methods
- Keep methods under 20 lines where possible
- Use meaningful variable names
- Prefer composition over inheritance
