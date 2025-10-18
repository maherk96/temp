```java

import quickfix.FieldNotFound;
import quickfix.Message;

private static String safeTag(Message msg, int tag) {
    try {
        if (msg.isSetField(tag)) return msg.getString(tag);              // body
        if (msg.getHeader().isSetField(tag)) return msg.getHeader().getString(tag);
        if (msg.getTrailer().isSetField(tag)) return msg.getTrailer().getString(tag);
    } catch (FieldNotFound ignore) {
        // fall through
    }
    return "N/A";
}

static Stream<Arguments> nosFusionAlgoMessages() {
    return readFixMessageFromFile("fix_messages.txt").stream()
        .map(msg -> Arguments.of(
            safeTag(msg, 11),   // ClOrdID
            msg
        ));
}
```
