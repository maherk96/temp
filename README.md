```java
@Override
public void onMessage(Message message) {
    log.info("Received message: {}", message.getContent());

    var rawFixMessage = message.getContent();

    // STEP 1️⃣: Normalize delimiters (convert literal "SOH" or "^A" to ASCII 0x01)
    String fixReady = rawFixMessage
            .replace("SOH", "\u0001")
            .replace("^A", "\u0001")
            .trim();

    // STEP 2️⃣: Remove duplicate SOH delimiters (e.g. multiple \u0001 in a row)
    fixReady = fixReady.replaceAll("(\u0001){2,}", "\u0001");

    // STEP 3️⃣: Ensure message starts with 8=FIX
    int fixStart = fixReady.indexOf("8=FIX");
    if (fixStart > 0) {
        fixReady = fixReady.substring(fixStart);
    }

    // STEP 4️⃣: Remove non-printable or stray whitespace characters
    fixReady = fixReady.replaceAll("[\\r\\n\\t]+", "");

    // STEP 5️⃣: (Optional) Trim trailing SOH if present
    if (fixReady.endsWith("\u0001")) {
        fixReady = fixReady.substring(0, fixReady.length() - 1);
    }

    log.debug("Cleaned FIX message: {}", fixReady);

    // STEP 6️⃣: Continue processing safely
    processFixMessage(fixReady);
}

```
