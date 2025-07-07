```java
import java.time.Duration;
import java.util.Collection;
import java.util.List;
import java.util.Queue;
import java.util.function.Supplier;
import java.util.stream.Collectors;

import org.junit.jupiter.api.Assertions;
import lombok.extern.slf4j.Slf4j;

/**
 * Factory class providing access to CreditCheck mock message queues and
 * convenient wait utilities for verifying CreditCheckRequest and CreditCheckResponse messages.
 * <p>
 * Wraps a {@link CreditCheckMockMessageFactory} which holds the actual message queues.
 * Provides type-safe access to the queues and fluent wait methods for test assertions.
 */
@Slf4j
public record CreditCheckMockMessageWaitFactory(CreditCheckMockMessageFactory messageFactory) {

    private static final Duration DEFAULT_TIMEOUT = Duration.ofSeconds(10);

    /**
     * Gets the thread-safe queue that holds received {@link CreditCheckRequest} messages.
     *
     * @return the queue of CreditCheckRequest messages.
     */
    public Queue<CreditCheckRequest> getCreditCheckRequests() {
        return messageFactory.getCreditCheckRequestQueue();
    }

    /**
     * Gets the thread-safe queue that holds received {@link CreditCheckResponse} messages.
     *
     * @return the queue of CreditCheckResponse messages.
     */
    private Queue<CreditCheckResponse> getCreditCheckResponses() {
        return messageFactory.getCreditCheckResponseQueue();
    }

    /**
     * Waits for a {@link CreditCheckRequest} to arrive that matches the given base number.
     * Fails the test if no such request appears within the default timeout.
     *
     * @param baseNumber the expected base number.
     * @return the matching {@link CreditCheckRequest}.
     */
    public CreditCheckRequest waitForCreditCheckRequestWithBaseNumber(String baseNumber) {
        var criteria = List.of(new DescriptivePredicateBuilder<>(CreditCheckRequest.class)
            .equals(CreditCheckRequest::getBaseNumber, baseNumber, "BaseNumber")
            .build());

        return waitFor(this::getCreditCheckRequests, criteria,
            "CreditCheckRequest with base number [" + baseNumber + "] not found");
    }

    /**
     * Waits for a {@link CreditCheckResponse} to arrive whose nested header contains the given base number.
     * Fails the test if no such response appears within the default timeout.
     *
     * @param baseNumber the expected base number in the header.
     * @return the matching {@link CreditCheckResponse}.
     */
    public CreditCheckResponse waitForCreditCheckResponseWithBaseNumber(String baseNumber) {
        var criteria = List.of(new DescriptivePredicateBuilder<>(CreditCheckResponse.class)
            .equals(r -> r.getHeader().getBaseNum(), baseNumber, "BaseNumber")
            .build());

        return waitFor(this::getCreditCheckResponses, criteria,
            "CreditCheckResponse with base number [" + baseNumber + "] not found");
    }

    /**
     * Waits for any item in the given message source that matches all provided criteria.
     * If found, returns the first match; if the wait times out, the test fails.
     *
     * @param source         the message source (queue).
     * @param criteria       the list of {@link DescriptivePredicate} to match.
     * @param failureMessage the failure message if no match is found in time.
     * @param <T>            the type of message.
     * @return the first matching message.
     */
    private <T> T waitFor(
        Supplier<? extends Collection<T>> source,
        List<DescriptivePredicate<T>> criteria,
        String failureMessage
    ) {
        String conditionSummary = criteria.stream()
            .map(Object::toString)
            .collect(Collectors.joining(", "));

        log.info("Waiting for {} matching [{}] for up to {}",
            source.getClass().getSimpleName(),
            conditionSummary,
            DEFAULT_TIMEOUT
        );

        try {
            AwaitCriteria.waitForItemMatchingAllCriteria(
                source, criteria, DEFAULT_TIMEOUT, failureMessage
            );

            var match = source.get().stream()
                .filter(item -> criteria.stream().allMatch(p -> p.test(item)))
                .findFirst()
                .orElse(null);

            Assertions.assertNotNull(match, failureMessage);

            log.info("Found matching message [{}]: {}", conditionSummary, match);
            return match;
        } catch (WaitTimeOutException e) {
            Assertions.fail(e.getMessage());
            return null; // Unreachable but required for compile.
        }
    }
}
```
