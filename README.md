```java

@Override
public void onExchangeResponse(ExchangeResponse exchangeResponse) {
    var orderRef = exchangeResponse.getOrderRef();
    var responseType = exchangeResponse.getExchangeResponseType();

    log.info("Received exchange response for orderRef={}, responseType={}", orderRef, responseType);

    try {
        // Resolve session and order safely
        var sessionIdOpt = getSessionId(orderRef);
        var sessionId = sessionIdOpt.orElseThrow(() ->
            new IllegalStateException("Session ID not found for orderRef=" + orderRef));

        var orderOpt = getOrder(sessionId, orderRef);
        var order = orderOpt.orElseThrow(() ->
            new IllegalStateException("Order not found for sessionId=" + sessionId + ", orderRef=" + orderRef));

        log.info("Processing exchange response for sessionId={}, orderRef={}", sessionId, orderRef);

        // Always process the main execution report
        fixGatewayAdaptor.processExchangeExecutionReport(sessionId, exchangeResponse, order);

        // If this is a cancel/reject type, handle OCR-specific logic
        if (CANCELLED_RESPONSE_TYPES.contains(responseType)) {
            processCancelResponse(sessionId, orderRef, exchangeResponse);
        }

    } catch (Exception e) {
        log.error("Error processing exchange response for orderRef={}: {}", orderRef, e.getMessage(), e);
    }
}

/**
 * Handles cancel-related exchange responses.
 */
private void processCancelResponse(String sessionId, String orderRef, ExchangeResponse exchangeResponse) {
    orderManager.getCancelRequest(orderRef).ifPresentOrElse(
        ocr -> {
            log.info("Processing cancel-related response for sessionId={}, orderRef={}", sessionId, orderRef);
            fixGatewayAdaptor.processExchangeExecutionReport(sessionId, exchangeResponse, ocr);
        },
        () -> {
            var msg = String.format("Cancel request not found for sessionId=%s, orderRef=%s", sessionId, orderRef);
            log.error(msg);
            throw new IllegalStateException(msg);
        }
    );
}```
