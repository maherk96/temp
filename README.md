```java
private static final DateTimeFormatter DB_TIMESTAMP =
        DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

private String toDbTimestamp(LocalDate date) {
    return date.atStartOfDay().format(DB_TIMESTAMP);
}
```
