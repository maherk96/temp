```java
case DEFAULT -> {
    log.info("Processing order with default action: {} ({})", order.getOrderRef(), orderAction.name());

    // --- Step 1: Check liquidity before sending any NEW response ---
    var liquidityAtPrice = getLiquidityAtPrice(order.getSymbol(), exchangeSide, order.getPrice());
    log.info("Liquidity at price {} on inception: {}", order.getPrice(), liquidityAtPrice);

    var isFillOrKillUnfillable = isFillOrKillUnfillable(order, liquidityAtPrice);

    // --- Step 2: Reject FOK orders that cannot be filled ---
    if (isFillOrKillUnfillable) {
        log.info("Evaluating FOK for order {} with liquidity {} → cannot be filled, rejecting",
                 order.getOrderRef(), liquidityAtPrice);

        exchangeResponse.setExchangeResponseType(ExchangeResponseType.CANCEL);
        exchangeResponse.setOrderId(orderID);
        exchangeResponse.setOrigClOrdId(order.getClOrdId());
        exchangeResponse.setFilledAmount(0);
        exchangeResponse.setFillPrice(0);
        sendExchangeResponse();
        return;
    }

    // --- Step 3: Accept the order (NEW) once validation passes ---
    exchangeResponse.setExchangeResponseType(ExchangeResponseType.NEW);
    exchangeResponse.setOrderId(orderID);
    exchangeResponse.setOrigClOrdId(order.getClOrdId());
    sendExchangeResponse();

    // --- Step 4: If liquidity is available, fill what can be filled ---
    if (liquidityAtPrice > 0) {
        double fillAmount = Math.min(order.getAmount(), liquidityAtPrice);
        double fillPrice = getFilledPrice(order.getSymbol(), exchangeSide, fillAmount);

        exchangeResponse.setExchangeResponseType(ExchangeResponseType.FILL);
        exchangeResponse.setFilledAmount(fillAmount);
        exchangeResponse.setFillPrice(fillPrice);

        log.info("Order {} filled: amount={}, price={}", order.getOrderRef(), fillAmount, fillPrice);
        sendExchangeResponse();
    }

    // --- Step 5: If partially filled, add remaining amount to internal orders ---
    if (exchangeResponse.getFilledAmount() < order.getAmount()) {
        addInternalOrder(order, exchangeResponse.getFilledAmount());
        log.info("Order {} partially filled, adding to internal orders with filled amount {}",
                 order.getOrderRef(), exchangeResponse.getFilledAmount());
    }
}

fix(exchange-manager): evaluate FOK and liquidity before sending NEW response

Reordered DEFAULT case logic to prevent sending ExchangeResponseType.NEW 
before FOK validation. Previously, unfillable FOK orders were being 
activated and partially filled due to premature NEW responses.

Now, liquidity and FOK are checked first:
- If FOK cannot fill → CANCEL is sent immediately
- Only valid orders proceed to NEW/FILL flow
- Partial fills are properly handled afterward
```
