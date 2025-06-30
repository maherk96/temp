```java
public class EntityAssert<T> {

    private final T actual;

    private EntityAssert(T actual) {
        this.actual = actual;
    }

    public static <T> EntityAssert<T> assertThat(T actual) {
        return new EntityAssert<>(actual);
    }

    public EntityAssert<T> isNotNull() {
        if (actual == null) {
            throw new AssertionError("Expected entity to be non-null but was null");
        }
        return this;
    }

    public <R> EntityAssert<T> has(Function<T, R> getter, R expected) {
        R value = getter.apply(actual);
        if (value == null && expected == null) return this;
        if (value == null || !value.equals(expected)) {
            throw new AssertionError(
                "Expected: " + expected + " but was: " + value
            );
        }
        return this;
    }

    public <R> EntityAssert<T> isNotNull(Function<T, R> getter) {
        R value = getter.apply(actual);
        if (value == null) {
            throw new AssertionError("Expected value to be non-null but was null");
        }
        return this;
    }

    public EntityAssert<T> matches(Predicate<T> condition, String message) {
        if (!condition.test(actual)) {
            throw new AssertionError("Assertion failed: " + message);
        }
        return this;
    }

    public T get() {
        return actual;
    }
}

package com.example.asserts;

import java.util.List;
import java.util.function.Predicate;
import java.util.function.Consumer;

public class EntityListAssert<T> {

    private final List<T> actual;

    private EntityListAssert(List<T> actual) {
        this.actual = actual;
    }

    public static <T> EntityListAssert<T> assertThat(List<T> actual) {
        return new EntityListAssert<>(actual);
    }

    public EntityListAssert<T> hasRows(int expectedSize) {
        int size = actual.size();
        if (size != expectedSize) {
            throw new AssertionError("Expected size " + expectedSize + " but was " + size);
        }
        return this;
    }

    public EntityListAssert<T> isNotEmpty() {
        if (actual.isEmpty()) {
            throw new AssertionError("Expected list to be non-empty but was empty");
        }
        return this;
    }

    public EntityListAssert<T> allMatch(Predicate<T> condition, String message) {
        boolean allMatch = actual.stream().allMatch(condition);
        if (!allMatch) {
            throw new AssertionError("allMatch failed: " + message);
        }
        return this;
    }

    public EntityListAssert<T> anyMatch(Predicate<T> condition, String message) {
        boolean anyMatch = actual.stream().anyMatch(condition);
        if (!anyMatch) {
            throw new AssertionError("anyMatch failed: " + message);
        }
        return this;
    }

    public EntityListAssert<T> forEach(Consumer<T> assertion) {
        actual.forEach(assertion);
        return this;
    }

    public List<T> get() {
        return actual;
    }
}



```
