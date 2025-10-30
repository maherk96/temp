```java
/**
 * Process order with DEFAULT action
 */
private void processDefaultOrder(ExchangeOrder order, String orderID) {
    // Send NEW response
    sendNewResponse(orderID);

    ExchangeSide exchangeSide = getExchangeSide(order.getSide());
    double liquidityAtPrice = getLiquidityAtPrice(order.getSymbol(), exchangeSide, order.getPrice());
    log.info("Liquidity at price {} on inception: {}", order.getPrice(), liquidityAtPrice);

    // Check Fill-or-Kill constraint
    if (isFillOrKillUnfillable(order, liquidityAtPrice)) {
        rejectFillOrKillOrder(order);
        return;
    }

    // Attempt immediate fill
    if (liquidityAtPrice > 0) {
        fillOrder(order, exchangeSide, liquidityAtPrice);
    }

    double filledAmount = exchangeResponse.getFilledAmount();
    double remainingAmount = order.getAmount() - filledAmount;

    // Handle any unfilled quantity
    if (remainingAmount > 0) {
        // Handle IOC logic
        if (order.getTimeInForce() == ExchangeTimeInForce.IMMEDIATE_OR_CANCEL) {
            log.info(
                "Cancelling remaining {} units for IOC order {} (filled {} of {})",
                remainingAmount,
                order.getOrderRef(),
                filledAmount,
                order.getAmount()
            );
            exchangeResponse.setExchangeResponseType(ExchangeResponseType.CANCEL);
            exchangeResponse.setFilledAmount(filledAmount);
            exchangeResponse.setFillPrice(exchangeResponse.getFillPrice());
            sendExchangeResponse();
            return;
        }

        // Otherwise, rest the order on the book
        addInternalOrder(order, filledAmount);
        log.info(
            "Order {} partially filled, resting remainder (filled {}, remaining {})",
            order.getOrderRef(),
            filledAmount,
            remainingAmount
        );
    }
}


```
