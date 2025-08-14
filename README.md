```java
package com.example.common.util;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.databind.json.JsonMapper;
import com.fasterxml.jackson.databind.module.SimpleModule;
import com.fasterxml.jackson.core.json.JsonReadFeature;

import java.io.IOException;
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;

/**
 * Generic JSON utility using Jackson.
 * Supports comments, trailing commas, and case-insensitive fields.
 * Thread-safe and reusable across the application.
 */
public final class JsonUtil {

    private static final ObjectMapper MAPPER;

    static {
        MAPPER = JsonMapper.builder()
                .enable(MapperFeature.ACCEPT_CASE_INSENSITIVE_PROPERTIES)
                .enable(JsonReadFeature.ALLOW_JAVA_COMMENTS.mappedFeature())
                .enable(JsonReadFeature.ALLOW_TRAILING_COMMA.mappedFeature())
                .enable(JsonReadFeature.ALLOW_SINGLE_QUOTES.mappedFeature())
                .enable(JsonReadFeature.ALLOW_UNQUOTED_FIELD_NAMES.mappedFeature())
                .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)
                .enable(SerializationFeature.INDENT_OUTPUT)
                .build();

        // Custom serializers/deserializers can be registered here
        MAPPER.registerModule(new SimpleModule());
    }

    private JsonUtil() {
        // Prevent instantiation
    }

    /**
     * Reads JSON from file into the given class type.
     */
    public static <T> T read(Path path, Class<T> type) throws IOException {
        try (InputStream in = Files.newInputStream(path)) {
            return MAPPER.readValue(in, type);
        }
    }

    /**
     * Reads JSON from string into the given class type.
     */
    public static <T> T read(String json, Class<T> type) throws JsonProcessingException {
        return MAPPER.readValue(json, type);
    }

    /**
     * Reads JSON from file into a generic type.
     */
    public static <T> T read(Path path, JavaType type) throws IOException {
        try (InputStream in = Files.newInputStream(path)) {
            return MAPPER.readValue(in, type);
        }
    }

    /**
     * Reads JSON from string into a generic type.
     */
    public static <T> T read(String json, JavaType type) throws JsonProcessingException {
        return MAPPER.readValue(json, type);
    }

    /**
     * Writes an object to JSON file (pretty printed).
     */
    public static void write(Path path, Object obj) throws IOException {
        MAPPER.writeValue(path.toFile(), obj);
    }

    /**
     * Converts an object to a JSON string (pretty printed).
     */
    public static String toJson(Object obj) throws JsonProcessingException {
        return MAPPER.writerWithDefaultPrettyPrinter().writeValueAsString(obj);
    }

    /**
     * Returns the shared ObjectMapper instance.
     */
    public static ObjectMapper mapper() {
        return MAPPER;
    }
}

```
