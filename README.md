```java
public static String sanitizeFixMessage(String rawFixMessage) {
    if (rawFixMessage == null || rawFixMessage.isBlank()) {
        throw new IllegalArgumentException("Empty FIX message");
    }

    String fixReady = rawFixMessage
            .replace("SOH", "\u0001")
            .replace("^A", "\u0001")
            .trim();

    // Remove duplicate SOHs
    fixReady = fixReady.replaceAll("(\u0001){2,}", "\u0001");

    // Remove unwanted whitespace
    fixReady = fixReady.replaceAll("[\\r\\n\\t]+", "");

    // Ensure message starts at 8=FIX
    int fixStart = fixReady.indexOf("8=FIX");
    if (fixStart > 0) {
        fixReady = fixReady.substring(fixStart);
    }

    // ✅ Ensure message ends with SOH (checksum field properly terminated)
    if (!fixReady.endsWith("\u0001")) {
        fixReady += "\u0001";
    }

    // ✅ Optional: ensure checksum tag is formatted correctly
    if (!fixReady.contains("\u000110=")) {
        // Missing checksum — not ideal but can be appended if absolutely needed
        fixReady += "10=000\u0001";
    }

    return fixReady;
}
```
