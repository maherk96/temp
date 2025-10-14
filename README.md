```java

public final class FixDateMapper {
    private FixDateMapper() {}

    public static LocalDateTime toLocalDateTime(Object time) {
        if (time == null) return null;

        if (time instanceof LocalDateTime ldt) {
            return ldt;
        } else if (time instanceof Instant i) {
            return LocalDateTime.ofInstant(i, ZoneOffset.UTC);
        } else if (time instanceof String s) {
            return LocalDateTime.parse(s);
        } else {
            throw new IllegalArgumentException("Unsupported TransactTime type: " + time.getClass());
        }
    }
}
```
