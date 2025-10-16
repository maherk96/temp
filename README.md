```java

public void waitForExecutionReport(
        String orderId,
        List<COREexecType> execTypes,
        Duration timeout,
        String errorMessage) {

    // --- 1️⃣ Build predicates for each expected execType ---
    List<DescriptivePredicate<CorMockExecutionReport>> criteria = execTypes.stream()
            .map(execType -> new DescriptivePredicateBuilder<CorMockExecutionReport>()
                    .equals(CorMockExecutionReport::getClOrdLinkID, orderId, "clOrdID")
                    .equals(CorMockExecutionReport::getExecType, execType, "execType")
                    .buildAsPredicate())
            .collect(Collectors.toList());

    var conditions = criteria.stream()
            .map(DescriptivePredicate::toString)
            .collect(Collectors.joining(", "));

    log.info("Waiting for Execution Reports for clOrdLinkID={} and execTypes={} | Conditions: {}",
            orderId, execTypes, conditions);

    // --- 2️⃣ Await all expected reports ---
    AwaitCriteria.waitForAllCriteriaToMatchAcrossCollection(
            this::getCorExecutionReports,
            criteria,
            timeout,
            errorMessage);

    // --- 3️⃣ Fetch all reports after waiting ---
    var reports = getCorExecutionReports();

    // --- 4️⃣ Group results: which reports matched which predicates ---
    Map<DescriptivePredicate<CorMockExecutionReport>, List<CorMockExecutionReport>> matchesByPredicate =
            criteria.stream().collect(Collectors.toMap(
                    p -> p,
                    p -> reports.stream().filter(p).collect(Collectors.toList())
            ));

    // --- 5️⃣ Log per-predicate results ---
    matchesByPredicate.forEach((predicate, matched) -> {
        if (matched.isEmpty()) {
            log.warn("❌ No reports matched predicate [{}]", predicate);
        } else {
            log.info("✅ {} report(s) matched predicate [{}]", matched.size(), predicate);
            matched.forEach(r -> log.info(
                    "   ↳ clOrdId={}, execType={}, symbol={}, qty={}",
                    r.getClOrdLinkID(),
                    r.getExecType(),
                    r.getSymbol(),
                    r.getOrderQty()
            ));
        }
    });

    // --- 6️⃣ Summary across all predicates ---
    boolean allMatched = matchesByPredicate.values().stream().allMatch(list -> !list.isEmpty());

    if (allMatched) {
        log.info("✅ All {} expected execTypes found for orderId={}", execTypes.size(), orderId);
    } else {
        log.warn("❌ Missing one or more expected execTypes for orderId={}. Expected={}, Found={}",
                orderId,
                execTypes,
                matchesByPredicate.entrySet().stream()
                        .filter(e -> !e.getValue().isEmpty())
                        .map(e -> e.getKey().toString())
                        .collect(Collectors.toList()));
    }

    // --- 7️⃣ (Optional) Flatten matched reports for further use ---
    List<CorMockExecutionReport> matchedReports = matchesByPredicate.values().stream()
            .flatMap(Collection::stream)
            .distinct()
            .collect(Collectors.toList());

    log.info("Matched Execution Reports ({} total): {}", matchedReports.size(), matchedReports);
}
```
