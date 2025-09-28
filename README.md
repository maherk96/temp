```java
// ===== FixTag.java =====
/**
 * Enum for commonly used FIX tags for better readability and type safety
 */
public enum FixTag {
    ACCOUNT("1"),
    CLORD_ID("11"),
    EXEC_ID("17"),
    ORDER_QTY("38"),
    ORD_STATUS("39"),
    ORD_TYPE("40"),
    PRICE("44"),
    SIDE("54"),
    SYMBOL("55"),
    TIME_IN_FORCE("59"),
    TRANSACT_TIME("60"),
    EXEC_TYPE("150");

    private final String tag;

    FixTag(String tag) {
        this.tag = tag;
    }

    public String getTag() {
        return tag;
    }

    @Override
    public String toString() {
        return tag;
    }

    /**
     * Find FixTag by tag string
     */
    public static FixTag fromTag(String tag) {
        for (FixTag fixTag : values()) {
            if (fixTag.tag.equals(tag)) {
                return fixTag;
            }
        }
        return null;
    }
}

// ===== TemplateResolutionException.java =====
/**
 * Exception thrown when a template cannot be resolved
 */
public class TemplateResolutionException extends RuntimeException {
    
    private final String template;
    private final String variableName;

    public TemplateResolutionException(String message) {
        super(message);
        this.template = null;
        this.variableName = null;
    }

    public TemplateResolutionException(String message, String template) {
        super(message);
        this.template = template;
        this.variableName = null;
    }

    public TemplateResolutionException(String message, String template, String variableName) {
        super(message);
        this.template = template;
        this.variableName = variableName;
    }

    public TemplateResolutionException(String message, Throwable cause) {
        super(message, cause);
        this.template = null;
        this.variableName = null;
    }

    public String getTemplate() {
        return template;
    }

    public String getVariableName() {
        return variableName;
    }
}

// ===== RandomValueGenerator.java =====
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.List;
import java.util.Map;
import java.util.Random;
import java.util.concurrent.ThreadLocalRandom;

/**
 * Handles generation of random values for FIX message fields
 */
public class RandomValueGenerator {
    
    private static final Logger logger = LoggerFactory.getLogger(RandomValueGenerator.class);
    
    private final Random random;

    /**
     * Default constructor using ThreadLocalRandom for production
     */
    public RandomValueGenerator() {
        this.random = ThreadLocalRandom.current();
    }

    /**
     * Constructor with injectable Random for deterministic testing
     */
    public RandomValueGenerator(Random random) {
        this.random = random;
    }

    /**
     * Get random item from a list variable
     */
    @SuppressWarnings("unchecked")
    public String randomFrom(String varName, Map<String, Object> vars, String defaultValue) {
        if (vars == null) {
            logger.warn("Variables map is null, using default value: {}", defaultValue);
            return defaultValue;
        }

        Object varValue = vars.get(varName);
        if (varValue == null) {
            if (defaultValue != null) {
                logger.debug("Variable '{}' not found, using default: {}", varName, defaultValue);
                return defaultValue;
            }
            throw new TemplateResolutionException("Variable not found: " + varName, "randomFrom:" + varName, varName);
        }

        if (varValue instanceof List) {
            List<Object> list = (List<Object>) varValue;
            if (list.isEmpty()) {
                if (defaultValue != null) {
                    logger.debug("Variable '{}' is empty list, using default: {}", varName, defaultValue);
                    return defaultValue;
                }
                throw new TemplateResolutionException("Variable list is empty: " + varName, "randomFrom:" + varName, varName);
            }
            
            Object randomItem = list.get(random.nextInt(list.size()));
            return randomItem.toString();
        } else {
            throw new TemplateResolutionException("Variable is not a list: " + varName, "randomFrom:" + varName, varName);
        }
    }

    /**
     * Generate random price from a range variable
     */
    public BigDecimal randomPrice(String varName, Map<String, Object> vars, BigDecimal defaultValue) {
        return randomDecimal(varName, vars, defaultValue, 2);
    }

    /**
     * Generate random quantity from a range variable
     */
    public BigDecimal randomQuantity(String varName, Map<String, Object> vars, BigDecimal defaultValue) {
        return randomDecimal(varName, vars, defaultValue, 0);
    }

    /**
     * Generate random integer from a range variable
     */
    public int randomInt(String varName, Map<String, Object> vars, Integer defaultValue) {
        BigDecimal result = randomDecimal(varName, vars, 
            defaultValue != null ? BigDecimal.valueOf(defaultValue) : BigDecimal.ONE, 0);
        return result.intValue();
    }

    /**
     * Generate random decimal from a range variable
     */
    @SuppressWarnings("unchecked")
    private BigDecimal randomDecimal(String varName, Map<String, Object> vars, BigDecimal defaultValue, int scale) {
        if (vars == null) {
            logger.warn("Variables map is null, using default value: {}", defaultValue);
            return defaultValue;
        }

        Object varValue = vars.get(varName);
        if (varValue == null) {
            if (defaultValue != null) {
                logger.debug("Variable '{}' not found, using default: {}", varName, defaultValue);
                return defaultValue;
            }
            throw new TemplateResolutionException("Variable not found: " + varName, "randomDecimal:" + varName, varName);
        }

        if (varValue instanceof Map) {
            Map<String, Object> range = (Map<String, Object>) varValue;
            
            BigDecimal min = parseDecimal(range.get("min"), BigDecimal.ONE);
            BigDecimal max = parseDecimal(range.get("max"), BigDecimal.valueOf(100));
            
            if (min.compareTo(max) > 0) {
                throw new TemplateResolutionException(
                    String.format("Invalid range: min (%s) > max (%s) for variable: %s", min, max, varName),
                    "randomDecimal:" + varName, varName);
            }

            // Generate random decimal between min and max
            double randomDouble = min.doubleValue() + 
                (random.nextDouble() * (max.subtract(min).doubleValue()));
                
            return BigDecimal.valueOf(randomDouble).setScale(scale, RoundingMode.HALF_UP);
        } else {
            throw new TemplateResolutionException("Variable is not a range object: " + varName, "randomDecimal:" + varName, varName);
        }
    }

    /**
     * Parse object to BigDecimal with fallback
     */
    private BigDecimal parseDecimal(Object value, BigDecimal defaultValue) {
        if (value == null) {
            return defaultValue;
        }
        
        if (value instanceof Number) {
            return BigDecimal.valueOf(((Number) value).doubleValue());
        }
        
        if (value instanceof String) {
            try {
                return new BigDecimal((String) value);
            } catch (NumberFormatException e) {
                logger.warn("Cannot parse '{}' as decimal, using default: {}", value, defaultValue);
                return defaultValue;
            }
        }
        
        logger.warn("Cannot convert '{}' to decimal, using default: {}", value, defaultValue);
        return defaultValue;
    }
}

// ===== TemplateResolver.java =====
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.Map;
import java.util.Random;
import java.util.UUID;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * Resolves template expressions and variables from GlobalConfig
 */
public class TemplateResolver {
    
    private static final Logger logger = LoggerFactory.getLogger(TemplateResolver.class);
    
    private static final Pattern TEMPLATE_PATTERN = Pattern.compile("\\{\\{([^}]+)\\}\\}");
    private static final Pattern PARAM_PATTERN = Pattern.compile("([^:]+):([^:]+)(?::default=(.+))?");
    
    private final RandomValueGenerator randomValueGenerator;

    public TemplateResolver() {
        this.randomValueGenerator = new RandomValueGenerator();
    }

    public TemplateResolver(Random random) {
        this.randomValueGenerator = new RandomValueGenerator(random);
    }

    /**
     * Resolve a field value that may contain templates
     */
    public Object resolveValue(String fieldValue, Map<String, Object> vars) {
        if (fieldValue == null || fieldValue.trim().isEmpty()) {
            return "";
        }

        // Check if it's a template (starts with {{ and ends with }})
        if (isTemplate(fieldValue)) {
            return resolveTemplate(fieldValue, vars);
        }

        // Direct value - return as-is
        return fieldValue;
    }

    /**
     * Check if a value is a template expression
     */
    private boolean isTemplate(String value) {
        return value.startsWith("{{") && value.endsWith("}}");
    }

    /**
     * Resolve template expressions
     */
    private Object resolveTemplate(String template, Map<String, Object> vars) {
        Matcher matcher = TEMPLATE_PATTERN.matcher(template);
        
        if (!matcher.find()) {
            throw new TemplateResolutionException("Invalid template format: " + template, template);
        }

        String templateContent = matcher.group(1).trim();
        logger.debug("Resolving template: {}", templateContent);

        // Handle built-in templates
        switch (templateContent) {
            case "randomOrderId":
                return generateRandomOrderId();
                
            case "currentTime":
                return LocalDateTime.now();
                
            case "uuid":
                return UUID.randomUUID().toString();
                
            default:
                return resolveParameterizedTemplate(templateContent, vars, template);
        }
    }

    /**
     * Resolve parameterized templates with optional defaults
     */
    private Object resolveParameterizedTemplate(String templateContent, Map<String, Object> vars, String originalTemplate) {
        Matcher paramMatcher = PARAM_PATTERN.matcher(templateContent);
        
        if (!paramMatcher.matches()) {
            throw new TemplateResolutionException("Invalid parameterized template format: " + templateContent, originalTemplate);
        }

        String operation = paramMatcher.group(1);
        String varName = paramMatcher.group(2);
        String defaultValue = paramMatcher.group(3); // Can be null

        logger.debug("Resolving parameterized template - operation: {}, varName: {}, default: {}", 
                    operation, varName, defaultValue);

        switch (operation) {
            case "randomFrom":
                return randomValueGenerator.randomFrom(varName, vars, defaultValue);
                
            case "randomPrice":
                return randomValueGenerator.randomPrice(varName, vars, 
                    defaultValue != null ? java.math.BigDecimal.valueOf(Double.parseDouble(defaultValue)) : java.math.BigDecimal.valueOf(100.0));
                
            case "randomQuantity":
                return randomValueGenerator.randomQuantity(varName, vars, 
                    defaultValue != null ? java.math.BigDecimal.valueOf(Double.parseDouble(defaultValue)) : java.math.BigDecimal.valueOf(100));
                
            case "randomInt":
                return randomValueGenerator.randomInt(varName, vars, 
                    defaultValue != null ? Integer.parseInt(defaultValue) : 1);
                
            case "currentTime":
                // Extended currentTime with format parameter
                return formatCurrentTime(varName); // varName is actually the format in this case
                
            default:
                throw new TemplateResolutionException("Unknown template operation: " + operation, originalTemplate, varName);
        }
    }

    /**
     * Generate random order ID
     */
    private String generateRandomOrderId() {
        return "ORD_" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
    }

    /**
     * Format current time with specified pattern
     */
    private String formatCurrentTime(String pattern) {
        try {
            DateTimeFormatter formatter = DateTimeFormatter.ofPattern(pattern);
            return LocalDateTime.now().format(formatter);
        } catch (Exception e) {
            logger.warn("Invalid date format pattern '{}', using default ISO format", pattern);
            return LocalDateTime.now().toString();
        }
    }

    /**
     * Resolve all templates in a map of field values
     */
    public Map<String, Object> resolveAllValues(Map<String, String> fields, Map<String, Object> vars) {
        Map<String, Object> resolvedFields = new java.util.HashMap<>();
        
        for (Map.Entry<String, String> entry : fields.entrySet()) {
            try {
                Object resolvedValue = resolveValue(entry.getValue(), vars);
                resolvedFields.put(entry.getKey(), resolvedValue);
                logger.debug("Resolved field {}: '{}' -> '{}'", entry.getKey(), entry.getValue(), resolvedValue);
            } catch (TemplateResolutionException e) {
                logger.error("Failed to resolve template for field {}: {}", entry.getKey(), e.getMessage());
                throw e;
            }
        }
        
        return resolvedFields;
    }

    /**
     * Check if a value contains any templates
     */
    public boolean containsTemplates(String value) {
        if (value == null) return false;
        return TEMPLATE_PATTERN.matcher(value).find();
    }

    /**
     * Validate that all required variables exist for the given fields
     */
    public void validateRequiredVariables(Map<String, String> fields, Map<String, Object> vars) {
        for (Map.Entry<String, String> entry : fields.entrySet()) {
            String fieldValue = entry.getValue();
            if (containsTemplates(fieldValue)) {
                validateTemplateVariables(fieldValue, vars, entry.getKey());
            }
        }
    }

    /**
     * Validate that variables required by a template exist
     */
    private void validateTemplateVariables(String template, Map<String, Object> vars, String fieldName) {
        Matcher matcher = TEMPLATE_PATTERN.matcher(template);
        
        while (matcher.find()) {
            String templateContent = matcher.group(1).trim();
            
            // Skip built-in templates
            if (isBuiltInTemplate(templateContent)) {
                continue;
            }
            
            Matcher paramMatcher = PARAM_PATTERN.matcher(templateContent);
            if (paramMatcher.matches()) {
                String varName = paramMatcher.group(2);
                String defaultValue = paramMatcher.group(3);
                
                // If no default value, the variable must exist
                if (defaultValue == null && (vars == null || !vars.containsKey(varName))) {
                    throw new TemplateResolutionException(
                        String.format("Required variable '%s' not found for field '%s'", varName, fieldName),
                        template, varName);
                }
            }
        }
    }

    /**
     * Check if template is a built-in template that doesn't require variables
     */
    private boolean isBuiltInTemplate(String templateContent) {
        return templateContent.equals("randomOrderId") || 
               templateContent.equals("currentTime") || 
               templateContent.equals("uuid");
    }
}

// ===== FixFieldMapper.java =====
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import quickfix.Message;
import quickfix.field.*;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.HashMap;
import java.util.Map;
import java.util.function.BiConsumer;

/**
 * Maps FIX tags to QuickFIX/J fields using a registry pattern
 */
public class FixFieldMapper {
    
    private static final Logger logger = LoggerFactory.getLogger(FixFieldMapper.class);
    
    private final Map<String, BiConsumer<Message, Object>> fieldMappers;

    public FixFieldMapper() {
        this.fieldMappers = new HashMap<>();
        initializeFieldMappers();
    }

    /**
     * Initialize the field mapping registry
     */
    private void initializeFieldMappers() {
        // Standard FIX 4.4 fields
        registerField(FixTag.ACCOUNT.getTag(), this::setAccount);
        registerField(FixTag.CLORD_ID.getTag(), this::setClOrdID);
        registerField(FixTag.EXEC_ID.getTag(), this::setExecID);
        registerField(FixTag.ORDER_QTY.getTag(), this::setOrderQty);
        registerField(FixTag.ORD_STATUS.getTag(), this::setOrdStatus);
        registerField(FixTag.ORD_TYPE.getTag(), this::setOrdType);
        registerField(FixTag.PRICE.getTag(), this::setPrice);
        registerField(FixTag.SIDE.getTag(), this::setSide);
        registerField(FixTag.SYMBOL.getTag(), this::setSymbol);
        registerField(FixTag.TIME_IN_FORCE.getTag(), this::setTimeInForce);
        registerField(FixTag.TRANSACT_TIME.getTag(), this::setTransactTime);
        registerField(FixTag.EXEC_TYPE.getTag(), this::setExecType);
        
        logger.debug("Initialized {} field mappers", fieldMappers.size());
    }

    /**
     * Register a new field mapper
     */
    public void registerField(String tag, BiConsumer<Message, Object> mapper) {
        fieldMappers.put(tag, mapper);
        logger.debug("Registered field mapper for tag: {}", tag);
    }

    /**
     * Set a field on a FIX message
     */
    public void setField(Message message, String tag, Object value) {
        if (value == null) {
            logger.debug("Skipping null value for tag: {}", tag);
            return;
        }

        BiConsumer<Message, Object> mapper = fieldMappers.get(tag);
        if (mapper != null) {
            try {
                mapper.accept(message, value);
                logger.debug("Set field {} = {}", tag, value);
            } catch (Exception e) {
                logger.error("Error setting field {} with value '{}': {}", tag, value, e.getMessage());
                // Fallback to setString
                setFieldAsString(message, tag, value);
            }
        } else {
            logger.warn("No specific mapper for tag {}, using string fallback", tag);
            setFieldAsString(message, tag, value);
        }
    }

    /**
     * Set all fields from a map
     */
    public void setAllFields(Message message, Map<String, Object> fields) {
        fields.forEach((tag, value) -> setField(message, tag, value));
    }

    /**
     * Fallback method to set field as string
     */
    private void setFieldAsString(Message message, String tag, Object value) {
        try {
            int tagInt = Integer.parseInt(tag);
            message.setString(tagInt, value.toString());
            logger.debug("Set field {} as string = {}", tag, value);
        } catch (NumberFormatException e) {
            logger.error("Invalid tag format: {}", tag);
        } catch (Exception e) {
            logger.error("Failed to set field {} as string: {}", tag, e.getMessage());
        }
    }

    // Individual field setters
    private void setAccount(Message message, Object value) {
        message.set(new Account(value.toString()));
    }

    private void setClOrdID(Message message, Object value) {
        message.set(new ClOrdID(value.toString()));
    }

    private void setExecID(Message message, Object value) {
        message.set(new ExecID(value.toString()));
    }

    private void setOrderQty(Message message, Object value) {
        BigDecimal qty = convertToBigDecimal(value);
        message.set(new OrderQty(qty.doubleValue()));
    }

    private void setOrdStatus(Message message, Object value) {
        char status = getCharValue(value);
        message.set(new OrdStatus(status));
    }

    private void setOrdType(Message message, Object value) {
        char type = getCharValue(value);
        message.set(new OrdType(type));
    }

    private void setPrice(Message message, Object value) {
        BigDecimal price = convertToBigDecimal(value);
        message.set(new Price(price.doubleValue()));
    }

    private void setSide(Message message, Object value) {
        char side = getCharValue(value);
        message.set(new Side(side));
    }

    private void setSymbol(Message message, Object value) {
        message.set(new Symbol(value.toString()));
    }

    private void setTimeInForce(Message message, Object value) {
        char tif = getCharValue(value);
        message.set(new TimeInForce(tif));
    }

    private void setTransactTime(Message message, Object value) {
        LocalDateTime dateTime = convertToLocalDateTime(value);
        message.set(new TransactTime(dateTime));
    }

    private void setExecType(Message message, Object value) {
        char execType = getCharValue(value);
        message.set(new ExecType(execType));
    }

    // Type conversion utilities
    private BigDecimal convertToBigDecimal(Object value) {
        if (value instanceof BigDecimal) {
            return (BigDecimal) value;
        }
        if (value instanceof Number) {
            return BigDecimal.valueOf(((Number) value).doubleValue());
        }
        if (value instanceof String) {
            try {
                return new BigDecimal((String) value);
            } catch (NumberFormatException e) {
                throw new IllegalArgumentException("Cannot convert to BigDecimal: " + value);
            }
        }
        throw new IllegalArgumentException("Cannot convert to BigDecimal: " + value);
    }

    private char getCharValue(Object value) {
        if (value instanceof Character) {
            return (Character) value;
        }
        String str = value.toString();
        if (str.length() != 1) {
            throw new IllegalArgumentException("Expected single character, got: " + str);
        }
        return str.charAt(0);
    }

    private LocalDateTime convertToLocalDateTime(Object value) {
        if (value instanceof LocalDateTime) {
            return (LocalDateTime) value;
        }
        if (value instanceof String) {
            String str = (String) value;
            
            // Try to parse ISO format first
            try {
                return LocalDateTime.parse(str);
            } catch (Exception e) {
                // Try FIX format: yyyyMMdd-HH:mm:ss.SSS
                try {
                    DateTimeFormatter fixFormatter = DateTimeFormatter.ofPattern("yyyyMMdd-HH:mm:ss.SSS");
                    return LocalDateTime.parse(str, fixFormatter);
                } catch (Exception e2) {
                    logger.warn("Cannot parse datetime '{}', using current time", str);
                    return LocalDateTime.now();
                }
            }
        }
        
        // Default to current time
        return LocalDateTime.now();
    }

    /**
     * Get all registered field tags
     */
    public java.util.Set<String> getRegisteredTags() {
        return fieldMappers.keySet();
    }

    /**
     * Check if a tag has a registered mapper
     */
    public boolean hasMapper(String tag) {
        return fieldMappers.containsKey(tag);
    }
}

// ===== JsonToFixMapper.java =====
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import quickfix.Message;
import quickfix.fix44.ExecutionReport;
import quickfix.fix44.NewOrderSingle;

import java.util.Map;
import java.util.Random;

/**
 * Orchestrates the creation of FIX messages from JSON request templates
 * Uses modular components for template resolution, random value generation, and field mapping
 */
public class JsonToFixMapper {
    
    private static final Logger logger = LoggerFactory.getLogger(JsonToFixMapper.class);
    
    private final TemplateResolver templateResolver;
    private final FixFieldMapper fixFieldMapper;

    /**
     * Default constructor for production use
     */
    public JsonToFixMapper() {
        this.templateResolver = new TemplateResolver();
        this.fixFieldMapper = new FixFieldMapper();
        logger.debug("JsonToFixMapper initialized with default components");
    }

    /**
     * Constructor with injectable Random for deterministic testing
     */
    public JsonToFixMapper(Random random) {
        this.templateResolver = new TemplateResolver(random);
        this.fixFieldMapper = new FixFieldMapper();
        logger.debug("JsonToFixMapper initialized with custom Random for testing");
    }

    /**
     * Create FIX message from Request using GlobalConfig for variable resolution
     */
    public Message createFixMessage(Request request, GlobalConfig globalConfig) {
        if (request == null) {
            throw new IllegalArgumentException("Request cannot be null");
        }
        
        if (request.getMessageType() == null) {
            throw new IllegalArgumentException("Message type cannot be null");
        }

        logger.debug("Creating FIX message of type: {}", request.getMessageType());

        // Validate that all required variables exist before processing
        Map<String, Object> vars = globalConfig != null ? globalConfig.getVars() : null;
        if (request.getFields() != null) {
            templateResolver.validateRequiredVariables(request.getFields(), vars);
        }

        // Create the appropriate message type
        Message message = createMessageByType(request.getMessageType());
        
        // Resolve templates and set fields
        if (request.getFields() != null && !request.getFields().isEmpty()) {
            Map<String, Object> resolvedFields = templateResolver.resolveAllValues(request.getFields(), vars);
            fixFieldMapper.setAllFields(message, resolvedFields);
            
            logger.debug("Successfully created {} with {} fields", 
                request.getMessageType(), resolvedFields.size());
        } else {
            logger.warn("No fields specified for message type: {}", request.getMessageType());
        }

        return message;
    }

    /**
     * Create message instance based on message type
     */
    private Message createMessageByType(FixMessageType messageType) {
        switch (messageType) {
            case NEW_ORDER_SINGLE:
                return new NewOrderSingle();
            
            case EXECUTION_REPORT:
                return new ExecutionReport();
            
            default:
                throw new IllegalArgumentException("Unsupported message type: " + messageType);
        }
    }

    /**
     * Register a custom field mapper
     */
    public void registerCustomField(String tag, java.util.function.BiConsumer<Message, Object> mapper) {
        fixFieldMapper.registerField(tag, mapper);
        logger.debug("Registered custom field mapper for tag: {}", tag);
    }

    /**
     * Demonstration method showing complete usage
     */
    public static void demonstrateUsage() {
        logger.info("=== JsonToFixMapper Demonstration ===");

        // Create mapper
        JsonToFixMapper mapper = new JsonToFixMapper();

        // Setup GlobalConfig with variables
        GlobalConfig globalConfig = new GlobalConfig();
        globalConfig.setConfigPath("fix-client-config.json");
        globalConfig.setEnvironmentName("ALGO_UAT");
        globalConfig.setClientStreamNames(java.util.Arrays.asList("CLIENT_001", "CLIENT_002"));

        // Setup vars
        Map<String, Object> vars = new java.util.HashMap<>();
        vars.put("symbols", java.util.Arrays.asList("AAPL", "GOOGL", "MSFT", "TSLA", "AMZN"));
        vars.put("sides", java.util.Arrays.asList("1", "2"));
        vars.put("orderTypes", java.util.Arrays.asList("1", "2", "3", "4"));
        vars.put("timeInForce", java.util.Arrays.asList("0", "1", "3", "4"));
        vars.put("accounts", java.util.Arrays.asList("ACC001", "ACC002", "ACC003", "ACC004"));

        Map<String, Object> priceRange = new java.util.HashMap<>();
        priceRange.put("min", 50.0);
        priceRange.put("max", 500.0);
        vars.put("priceRange", priceRange);

        Map<String, Object> quantityRange = new java.util.HashMap<>();
        quantityRange.put("min", 100);
        quantityRange.put("max", 10000);
        vars.put("quantityRange", quantityRange);

        globalConfig.setVars(vars);

        // Create NewOrderSingle request
        Request orderRequest = new Request();
        orderRequest.setMessageType(FixMessageType.NEW_ORDER_SINGLE);

        Map<String, String> orderFields = new java.util.HashMap<>();
        orderFields.put(FixTag.CLORD_ID.getTag(), "{{randomOrderId}}");
        orderFields.put(FixTag.SYMBOL.getTag(), "{{randomFrom:symbols}}");
        orderFields.put(FixTag.SIDE.getTag(), "{{randomFrom:sides}}");
        orderFields.put(FixTag.PRICE.getTag(), "{{randomPrice:priceRange}}");
        orderFields.put(FixTag.ORDER_QTY.getTag(), "{{randomQuantity:quantityRange}}");
        orderFields.put(FixTag.ORD_TYPE.getTag(), "2"); // Direct value: Limit
        orderFields.put(FixTag.TIME_IN_FORCE.getTag(), "{{randomFrom:timeInForce}}");
        orderFields.put(FixTag.ACCOUNT.getTag(), "{{randomFrom:accounts}}");
        orderFields.put(FixTag.TRANSACT_TIME.getTag(), "{{currentTime}}");

        orderRequest.setFields(orderFields);

        try {
            // Create message
            Message fixMessage = mapper.createFixMessage(orderRequest, globalConfig);
            logger.info("Generated FIX Message: {}", fixMessage.toString());
            
            // Print key fields for verification
            try {
                logger.info("  ClOrdID: {}", fixMessage.getString(quickfix.field.ClOrdID.FIELD));
                logger.info("  Symbol: {}", fixMessage.getString(quickfix.field.Symbol.FIELD));
                logger.info("  Side: {}", fixMessage.getString(quickfix.field.Side.FIELD));
                logger.info("  Price: {}", fixMessage.getString(quickfix.field.Price.FIELD));
                logger.info("  OrderQty: {}", fixMessage.getString(quickfix.field.OrderQty.FIELD));
            } catch (Exception e) {
                logger.warn("Could not extract field details: {}", e.getMessage());
            }

        } catch (Exception e) {
            logger.error("Demonstration failed: {}", e.getMessage(), e);
        }

        logger.info("=== Demonstration Complete ===");
    }

    /**
     * Main method for standalone testing
     */
    public static void main(String[] args) {
        demonstrateUsage();
    }
}

// ===== Usage Example Integration ===== 
/**
 * Example showing integration with your pool manager
 */
public class PoolManagerIntegrationExample {
    
    public static void main(String[] args) {
        // Create mapper and pool manager
        JsonToFixMapper mapper = new JsonToFixMapper();
        
        // Your existing pool manager setup
        FixClientPoolManager poolManager = new FixClientPoolManager(
            "fix-client-config.json", 
            "ALGO_UAT", 
            Set.of("CLIENT_001", "CLIENT_002")
        );
        
        // Setup configuration
        GlobalConfig config = createTestConfig();
        Request request = createOrderRequest();
        
        try {
            // Start pool
            poolManager.startAll();
            
            // Create FIX message from template
            Message fixMessage = mapper.createFixMessage(request, config);
            
            // Send via round-robin
            String clientUsed = poolManager.sendTradeMessage(fixMessage);
            System.out.println("Order sent via client: " + clientUsed);
            
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            poolManager.stopAll();
        }
    }
    
    private static GlobalConfig createTestConfig() {
        GlobalConfig config = new GlobalConfig();
        Map<String, Object> vars = new HashMap<>();
        vars.put("symbols", Arrays.asList("AAPL", "GOOGL", "MSFT"));
        vars.put("sides", Arrays.asList("1", "2"));
        
        Map<String, Object> priceRange = new HashMap<>();
        priceRange.put("min", 100.0);
        priceRange.put("max", 200.0);
        vars.put("priceRange", priceRange);
        
        config.setVars(vars);
        return config;
    }
    
    private static Request createOrderRequest() {
        Request request = new Request();
        request.setMessageType(FixMessageType.NEW_ORDER_SINGLE);
        
        Map<String, String> fields = new HashMap<>();
        fields.put("11", "{{randomOrderId}}");
        fields.put("55", "{{randomFrom:symbols}}");
        fields.put("54", "{{randomFrom:sides}}");
        fields.put("44", "{{randomPrice:priceRange}}");
        fields.put("38", "100"); // Direct quantity
        fields.put("40", "2");   // Direct order type
        
        request.setFields(fields);
        return request;
    }
}
```
