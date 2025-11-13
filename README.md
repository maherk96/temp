```java

private static final DateTimeFormatter FIX_SENDING_TIME_FORMAT =
        DateTimeFormatter.ofPattern("yyyyMMdd-HH:mm:ss.SSS")
                         .withZone(ZoneId.of("UTC"));

private static Instant parseSendingTime(String sendingTime) {
    return FIX_SENDING_TIME_FORMAT.parse(sendingTime, Instant::from);
}
```
