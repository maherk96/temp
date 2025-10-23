```java
case DEFAULT -> {
    log.info("Processing order with default action: {} ({})", order.getOrderRef(), orderAction.name());

    // get liquidity at the price level
    var liquidityAtPrice = getLiquidityAtPrice(order.getSymbol(), exchangeSide, order.getPrice());
    log.info("Liquidity at price {} on inception: {}", order.getPrice(), liquidityAtPrice);

    // check if order cannot be filled (for FOK)
    var isFillOrKillUnfillable = isFillOrKillUnfillable(order, liquidityAtPrice);

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

    // if order passes FOK validation, send NEW response
    exchangeResponse.setExchangeResponseType(ExchangeResponseType.NEW);
    exchangeResponse.setOrderId(orderID);
    exchangeResponse.setOrigClOrdId(order.getClOrdId());
    sendExchangeResponse();

    // continue normal flow after sending NEW
    log.info("Order {} accepted with liquidity {}", order.getOrderRef(), liquidityAtPrice);
    // additional order handling logic can follow here...
}

Previously, the DEFAULT case in the order handling logic sent an ExchangeResponseType.NEW 
response before checking Fill-or-Kill (FOK) conditions. This caused orders with zero liquidity 
to be incorrectly marked as active or partially filled, even though they should have been 
rejected immediately.

The bug occurred because the NEW response triggered downstream matching logic 
before FOK rejection was evaluated.

This fix reorders the logic so that:
- Liquidity and FOK validation are performed first.
- If the order cannot be filled (FOK unfillable), a CANCEL response is sent immediately.
- Only valid orders that pass validation send a NEW response.

As a result, FOK orders with insufficient liquidity are now properly rejected 
and no longer appear as partially filled in logs or downstream systems.
```
