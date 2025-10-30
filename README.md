```java


public class ExchangeImpl implements Exchange {
    private static final Logger log = LoggerFactory.getLogger(ExchangeImpl.class);

    /** The factor by which an order can be partially filled when filled automatically */
    private static final double PARTIAL_FILL_FACTOR = 0.5;

    /** The default rate at which resting orders are processed */
    private static final long DEFAULT_RESTING_ORDER_RATE = 500;

    /** The default time unit for resting order processing */
    private static final TimeUnit DEFAULT_RESTING_ORDER_TIME_UNIT = TimeUnit.MILLISECONDS;

    /** The default rate at which market data is published */
    private static final long DEFAULT_MARKET_DATA_PUBLISHER_RATE = 500;

    /** The default time unit for market data publishing */
    private static final TimeUnit DEFAULT_MARKET_DATA_PUBLISHER_TIME_UNIT = TimeUnit.MILLISECONDS;

    /** The default rate at which delayed orders are processed */
    private static final long DEFAULT_DELAYED_ORDER_RATE = 25;

    /** The default time unit for delayed order processing */
    private static final TimeUnit DEFAULT_DELAYED_ORDER_TIME_UNIT = TimeUnit.MILLISECONDS;

    /**
     * Live order tracker Orders are added when they come into the exchange to rest and are removed
     * when filled or cancelled
     */
    private final ExchangeOrderTracker liveOrders;

    /** Exchange response object to be sent back to the client */
    private final ExchangeResponse exchangeResponse = new ExchangeResponse();

    /** Market data object to be published by the exchange */
    private final ExchangeMarketData exchangeMarketData = new ExchangeMarketData();

    /** Order book container for the exchange */
    private final ExchangeOrderBookContainer exchangeOrderBookContainer;

    /** The name of the exchange */
    private final ExchangeName name;

    /** Price adjuster for the exchange */
    private final PriceAdjuster priceAdjuster;

    /** Timer for processing resting orders */
    private Timer restingOrdersTimer;

    /** Timer for market data publishing */
    private Timer marketDataPublisherTimer;

    /** Timer for processing delayed orders */
    private Timer delayedOrdersTimer;

    /** Market data publisher */
    private final ExchangeMarketDataPublisher marketDataPublisher;

    private final SingleExchangeCfg exchangeCfg;

    /** Order action set at the exchange level */
    private OrderAction orderAction;

    /** Exchange response listener */
    private final ExchangeResponseListener exchangeResponseListener;

    /** Delay controller for handling delayed fills */
    private final DelayController delayController = new DelayController();

    /** Sorted set for managing delayed exchange responses */
    private final TreeSet<DelayedExchangeResponse> delayedExchangeResponses = new TreeSet<>();

    /**
     * Constructor for the exchange
     *
     * @param name is the name of the exchange
     */
    public ExchangeImpl(
        ExchangeName name,
        ExchangeOrderTracker liveOrders,
        ExchangeOrderBookContainer exchangeOrderBookContainer,
        PriceAdjuster priceAdjuster,
        ExchangeResponseListener exchangeResponseListener,
        ExchangeMarketDataPublisher marketDataPublisher,
        SingleExchangeCfg exchangeCfg
    ) {
        this.name = name;
        this.liveOrders = liveOrders;
        this.exchangeOrderBookContainer = exchangeOrderBookContainer;
        this.priceAdjuster = priceAdjuster;
        this.exchangeResponseListener = exchangeResponseListener;
        this.exchangeResponse.setExchangeName(name);
        this.marketDataPublisher = marketDataPublisher;
        this.orderAction = exchangeCfg.getOrderAction();
        this.exchangeCfg = exchangeCfg;
    }

    private void sendExchangeResponse() {
        log.info("Sending exchange response: {}", exchangeResponse);
        exchangeResponseListener.onExchangeResponse(exchangeResponse);
    }

    private boolean isFillOrKillUnfillable(ExchangeOrder order, double liquidityAtPrice) {
        return order.getTimeInForce().equals(ExchangeTimeInForce.FILL_OR_KILL)
            && liquidityAtPrice < order.getAmount();
    }

    /**
     * Process orders coming into the exchange
     *
     * @param order is an order arriving at the exchange
     */
    @Override
    public void processOrder(ExchangeOrder order) {
        log.info("Exchange service received order {}", order);
        
        // Initialize exchange response
        initializeExchangeResponse(order);
        final String orderID = createExchangeOrderId();
        exchangeResponse.setOrderId(orderID);
        exchangeResponse.setOrigClOrdId(order.getClOrdId());
        
        // Send pending new acknowledgment
        sendPendingNew(order);
        
        // Determine effective order action
        OrderAction effectiveAction = determineEffectiveOrderAction(order);
        
        // Process based on order action
        processOrderByAction(order, orderID, effectiveAction);
    }

    /**
     * Initialize the exchange response with order details
     */
    private void initializeExchangeResponse(ExchangeOrder order) {
        exchangeResponse.setOrderRef(order.getOrderRef());
        exchangeResponse.setAmount(order.getAmount());
        exchangeResponse.setPrice(order.getPrice());
        exchangeResponse.setSide(order.getSide());
        exchangeResponse.setSymbol(order.getSymbol());
        exchangeResponse.setFilledAmount(0);
        exchangeResponse.setFillPrice(0);
    }

    /**
     * Send pending new response
     */
    private void sendPendingNew(ExchangeOrder order) {
        exchangeResponse.setExchangeResponseType(ExchangeResponseType.PENDING_NEW);
        log.info("Sending pending new response for order {}", order.getOrderRef());
        sendExchangeResponse();
    }

    /**
     * Determine the effective order action based on order and exchange configuration
     */
    private OrderAction determineEffectiveOrderAction(ExchangeOrder order) {
        return (this.orderAction == null || this.orderAction.equals(OrderAction.DEFAULT))
            ? order.getOrderAction()
            : this.orderAction;
    }

    /**
     * Process order based on the determined action
     */
    private void processOrderByAction(ExchangeOrder order, String orderID, OrderAction action) {
        log.info("Processing order with action: {} {}", order.getOrderRef(), action.name());
        
        switch (action) {
            case DEFAULT -> processDefaultOrder(order, orderID);
            case DELAY -> processDelayOrder(order, orderID);
            case FILL -> processFillOrder(order, orderID);
            case FILL_WITH_DELAY -> processFillWithDelayOrder(order, orderID);
            case CANCEL -> processCancelOrder(order, orderID);
            case NO_ACK -> processNoAckOrder(order);
            case PARTIAL_FILL -> processPartialFillOrder(order, orderID);
            case REJECT -> processRejectOrder(order, orderID);
        }
    }

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
        
        // Rest remaining quantity
        if (exchangeResponse.getFilledAmount() < order.getAmount()) {
            addInternalOrder(order, exchangeResponse.getFilledAmount());
            log.info("Order {} partially filled, resting remainder with filled amount {}",
                order.getOrderRef(), exchangeResponse.getFilledAmount());
        }
    }

    /**
     * Process order with DELAY action
     */
    private void processDelayOrder(ExchangeOrder order, String orderID) {
        // TODO: Implement DELAY action
        log.warn("DELAY action is not implemented yet for order: {}", order.getOrderRef());
    }

    /**
     * Process order with FILL action
     */
    private void processFillOrder(ExchangeOrder order, String orderID) {
        sendNewResponse(orderID);
        
        exchangeResponse.setExchangeResponseType(ExchangeResponseType.FILL);
        exchangeResponse.setFilledAmount(order.getAmount());
        exchangeResponse.setFillPrice(order.getPrice());
        log.info("Sending filled response for order {}", order.getOrderRef());
        sendExchangeResponse();
    }

    /**
     * Process order with FILL_WITH_DELAY action
     */
    private void processFillWithDelayOrder(ExchangeOrder order, String orderID) {
        sendNewResponse(orderID);
        
        long delay = delayController.calculateRandomDelay();
        TimeUnit timeUnit = DelayController.getTimeUnit();
        Instant scheduledDateTime = Instant.now().plusMillis(timeUnit.toMillis(delay));
        
        log.info("Delaying fill response for order {} by {} {} (scheduled for {})",
            order.getOrderRef(), delay, timeUnit, scheduledDateTime);
        
        exchangeResponse.setExchangeResponseType(ExchangeResponseType.FILL);
        exchangeResponse.setFilledAmount(order.getAmount());
        exchangeResponse.setFillPrice(order.getPrice());
        
        DelayedExchangeResponse delayedResponse = new DelayedExchangeResponse(
            exchangeResponse.getAmount(),
            exchangeResponse.getPrice(),
            exchangeResponse.getSymbol(),
            exchangeResponse.getSide(),
            exchangeResponse.getFilledAmount(),
            exchangeResponse.getOrderRef(),
            orderID,
            order.getClOrdId(),
            delay,
            exchangeResponse.getExchangeResponseType(),
            exchangeResponse.getExchangeName(),
            scheduledDateTime
        );
        insertDelayedExchangeResponse(delayedResponse);
    }

    /**
     * Process order with CANCEL action
     */
    private void processCancelOrder(ExchangeOrder order, String orderID) {
        sendNewResponse(orderID);
        
        exchangeResponse.setExchangeResponseType(ExchangeResponseType.CANCEL);
        exchangeResponse.setFilledAmount(0);
        exchangeResponse.setFillPrice(0);
        log.info("Sending cancel response for order {}", order.getOrderRef());
        sendExchangeResponse();
    }

    /**
     * Process order with NO_ACK action
     */
    private void processNoAckOrder(ExchangeOrder order) {
        log.info("Processing order with NO_ACK action: {}", order.getOrderRef());
        // No acknowledgment sent
    }

    /**
     * Process order with PARTIAL_FILL action
     */
    private void processPartialFillOrder(ExchangeOrder order, String orderID) {
        sendNewResponse(orderID);
        
        ExchangeSide exchangeSide = getExchangeSide(order.getSide());
        double partialAmount = order.getAmount() * PARTIAL_FILL_FACTOR;
        
        exchangeResponse.setExchangeResponseType(ExchangeResponseType.FILL);
        exchangeResponse.setFilledAmount(partialAmount);
        exchangeResponse.setFillPrice(
            getFilledPrice(order.getSymbol(), exchangeSide, partialAmount)
        );
        log.info("Sending partial fill response for order {} with filled amount {}",
            order.getOrderRef(), exchangeResponse.getFilledAmount());
        sendExchangeResponse();
        
        addInternalOrder(order, exchangeResponse.getFilledAmount());
        log.info("Order {} partially filled, adding to internal orders with filled amount {}",
            order.getOrderRef(), exchangeResponse.getFilledAmount());
    }

    /**
     * Process order with REJECT action
     */
    private void processRejectOrder(ExchangeOrder order, String orderID) {
        sendNewResponse(orderID);
        
        exchangeResponse.setExchangeResponseType(ExchangeResponseType.REJECT);
        exchangeResponse.setFilledAmount(0);
        exchangeResponse.setFillPrice(0);
        log.info("Sending reject response for order {}", order.getOrderRef());
        sendExchangeResponse();
    }

    /**
     * Send NEW response
     */
    private void sendNewResponse(String orderID) {
        exchangeResponse.setExchangeResponseType(ExchangeResponseType.NEW);
        exchangeResponse.setOrderId(orderID);
        sendExchangeResponse();
    }

    /**
     * Get exchange side from order side
     */
    private ExchangeSide getExchangeSide(Side side) {
        return side.equals(Side.BUY) ? ExchangeSide.OFFER : ExchangeSide.BID;
    }

    /**
     * Reject a Fill-or-Kill order that cannot be filled
     */
    private void rejectFillOrKillOrder(ExchangeOrder order) {
        log.info("Order {} cannot be filled (FOK), rejecting", order.getOrderRef());
        exchangeResponse.setExchangeResponseType(ExchangeResponseType.CANCEL);
        exchangeResponse.setFilledAmount(0);
        exchangeResponse.setFillPrice(0);
        sendExchangeResponse();
    }

    /**
     * Fill an order with available liquidity
     */
    private void fillOrder(ExchangeOrder order, ExchangeSide exchangeSide, double liquidityAtPrice) {
        log.info("Processing fill: LiquidityAtPrice={}, OrderAmount={}, FilledAmount={}",
            liquidityAtPrice, order.getAmount(), exchangeResponse.getFilledAmount());
        
        exchangeResponse.setExchangeResponseType(ExchangeResponseType.FILL);
        exchangeResponse.setFilledAmount(
            Math.min(order.getAmount(), liquidityAtPrice + exchangeResponse.getFilledAmount())
        );
        exchangeResponse.setFillPrice(
            getFilledPrice(order.getSymbol(), exchangeSide, exchangeResponse.getFilledAmount())
        );
        
        log.info("Order filled: FilledAmount={}, FillPrice={}",
            exchangeResponse.getFilledAmount(), exchangeResponse.getFillPrice());
        sendExchangeResponse();
    }

    /**
     * Inserts a DelayedExchangeResponse into the TreeSet
     *
     * @param response the delayed exchange response to insert
     */
    public void insertDelayedExchangeResponse(DelayedExchangeResponse response) {
        delayedExchangeResponses.add(response);
    }

    /**
     * Gets and removes all DelayedExchangeResponses from the TreeSet whose scheduledDateTime has
     * elapsed.
     *
     * @return a list of delayed exchange responses ready to be processed
     */
    public List<DelayedExchangeResponse> getAndRemoveNextDelayedExchangeResponses() {
        List<DelayedExchangeResponse> readyResponses = new ArrayList<>();
        Instant now = Instant.now();
        Iterator<DelayedExchangeResponse> iterator = delayedExchangeResponses.iterator();
        while (iterator.hasNext()) {
            DelayedExchangeResponse response = iterator.next();
            if (!response.getScheduledDateTime().isAfter(now)) {
                readyResponses.add(response);
                iterator.remove();
            } else {
                // TreeSet is ordered, so we can break early
                break;
            }
        }
        return readyResponses;
    }

    /**
     * Adds an internal order to the live orders
     *
     * @param order is the order to be added
     * @param filledAmount is the amount already filled
     */
    private void addInternalOrder(ExchangeOrder order, double filledAmount) {
        // store the order ref, symbol, side and amount in a map
        var internalOrder = new InternalOrder();
        internalOrder.setOrderRef(order.getOrderRef());
        internalOrder.setAmount(order.getAmount());
        internalOrder.setPrice(order.getPrice());
        internalOrder.setSymbol(order.getSymbol());
        internalOrder.setSide(order.getSide());
        internalOrder.setFilledAmount(filledAmount);

        log.debug("Adding internal order: {}", internalOrder);
        liveOrders.addOrder(order.getOrderRef(), internalOrder);
    }

    /**
     * Process a cancel request
     *
     * @param orderCancelRequest is the cancel request
     */
    @Override
    public void processCancel(ExchangeOrderCancelRequest orderCancelRequest) {
        var liveOrder = liveOrders.getOrder(orderCancelRequest.getOriginalOrderRef());
        exchangeResponse.setAmount(0);
        exchangeResponse.setPrice(0);
        exchangeResponse.setFilledAmount(0);
        exchangeResponse.setFillPrice(0);
        exchangeResponse.setOrderRef(orderCancelRequest.getOrderRef());
        exchangeResponse.setOrigClOrdId(orderCancelRequest.getOriginalOrderRef());
        exchangeResponse.setOrderId(createExchangeOrderId());
        exchangeResponse.setExchangeResponseType(ExchangeResponseType.PENDING_CANCEL);
        log.info("Processing cancel request for order {}", orderCancelRequest.getOrderRef());
        sendExchangeResponse();

        if (liveOrder != null) {
            exchangeResponse.setOrderRef(orderCancelRequest.getOrderRef());
            exchangeResponse.setAmount(liveOrder.getAmount());
            exchangeResponse.setPrice(liveOrder.getPrice());
            exchangeResponse.setSide(liveOrder.getSide());
            exchangeResponse.setSymbol(liveOrder.getSymbol());
            exchangeResponse.setExchangeResponseType(ExchangeResponseType.CANCEL);
            exchangeResponse.setFillPrice(0);
            exchangeResponse.setFilledAmount(0);
            liveOrders.removeOrder(liveOrder.getOrderRef());
            exchangeResponseListener.onExchangeResponse(exchangeResponse);
        }
    }

    /** Process all delayed orders - usually run from a timer task */
    @Override
    public void processAllDelayedOrders() {
        var delayedOrdersToProcess = getAndRemoveNextDelayedExchangeResponses();
        if (delayedOrdersToProcess.isEmpty()) {
            return;
        }

        log.debug("Processing delayed orders: {}", delayedOrdersToProcess);
        for (DelayedExchangeResponse delayedResponse : delayedOrdersToProcess) {
            exchangeResponseListener.onExchangeResponse(delayedResponse.getExchangeResponse());
        }
    }

    /** Process all resting orders - usually run from a timer task */
    @Override
    public void processAllRestingOrders() {
        liveOrders.getOpenOrders().forEach(this::processRestingOrder);
    }

    /**
     * Process an individual resting order
     *
     * @param order is the order
     */
    @Override
    public void processRestingOrder(InternalOrder order) {
        var exchangeSide = (order.getSide().equals(Side.BUY)) ? ExchangeSide.OFFER : ExchangeSide.BID;
        var liquidityAtPrice = getLiquidityAtPrice(order.getSymbol(), exchangeSide, order.getPrice());
        log.debug(
            "Liquidity at price for {} {}: {}",
            order.getSymbol(),
            exchangeSide,
            order.getPrice(),
            liquidityAtPrice
        );

        // check if the order can be partially or fully filled
        if (liquidityAtPrice > 0) {
            log.debug("Processing resting order: {}", order);
            exchangeResponse.setOrderRef(order.getOrderRef());
            exchangeResponse.setAmount(order.getAmount());
            exchangeResponse.setPrice(order.getPrice());
            exchangeResponse.setSide(order.getSide());
            exchangeResponse.setSymbol(order.getSymbol());
            var amountToFill = order.getAmount() - order.getFilledAmount();
            log.debug("Amount to fill: {}", amountToFill);

            exchangeResponse.setExchangeResponseType(ExchangeResponseType.FILL);
            exchangeResponse.setFilledAmount(Math.min(amountToFill, liquidityAtPrice + order.getFilledAmount()));
            exchangeResponse.setFillPrice(
                getFilledPrice(order.getSymbol(), exchangeSide, exchangeResponse.getFilledAmount())
            );

            if (exchangeResponse.getFilledAmount() < amountToFill) {
                order.setFilledAmount(order.getFilledAmount() + exchangeResponse.getFilledAmount());
            } else {
                liveOrders.removeOrder(order.getOrderRef());
            }

            log.debug("Sending exchange response: {}", exchangeResponse);
            exchangeResponseListener.onExchangeResponse(exchangeResponse);
        }
    }

    /**
     * Cancel an order
     *
     * @param orderRef is the order reference
     */
    @Override
    public void cancelOrder(String orderRef) {
        liveOrders.removeOrder(orderRef);
        exchangeResponse.setExchangeResponseType(ExchangeResponseType.REJECT);
        exchangeResponse.setFilledAmount(0);
        exchangeResponse.setFillPrice(0);
        exchangeResponseListener.onExchangeResponse(exchangeResponse);
    }

    /** Cancel all orders */
    @Override
    public void cancelAllOrders() {
        liveOrders
            .getOpenOrderSymbolsList()
            .forEach(symbol -> {
                liveOrders
                    .getOpenOrders()
                    .forEach(order -> cancelOrder(order.getOrderRef()));
            });
    }

    /**
     * Reject an order
     *
     * @param orderRef is the order reference
     */
    @Override
    public void rejectOrder(String orderRef) {
        // TODO write reject order code
    }

    /** Publish the order book */
    @Override
    public void publishOrderBook() {
        if (marketDataPublisher == null) {
            return;
        }

        // build up the book object
        exchangeOrderBookContainer
            .getExchangeOrderBook()
            .forEach((symbol, orderBookContainer) -> {
                // clear the lists of market data before processing
                exchangeMarketData.getBidPrices().clear();
                exchangeMarketData.getBidSizes().clear();
                exchangeMarketData.getOfferPrices().clear();
                exchangeMarketData.getOfferSizes().clear();
                exchangeMarketData.setExchangeName(name);
                exchangeMarketData.setSymbol(symbol);

                orderBookContainer
                    .getSymbolOrderBook()
                    .forEach((side, oneSidedSymbolOrderBook) -> {
                        var prices = (side.equals(ExchangeSide.BID))
                            ? exchangeMarketData.getBidPrices()
                            : exchangeMarketData.getOfferPrices();
                        var sizes = (side.equals(ExchangeSide.BID))
                            ? exchangeMarketData.getBidSizes()
                            : exchangeMarketData.getOfferSizes();

                        oneSidedSymbolOrderBook
                            .getOrderBook()
                            .forEach((price, amount) -> {
                                prices.add(price);
                                sizes.add(amount);
                            });
                    });

                marketDataPublisher.publishMarketData(exchangeMarketData, exchangeCfg);
            });

        log.debug(
            "Order Book published for exchange {}",
            exchangeOrderBookContainer.getExchangeOrderBook().toString()
        );
    }

    /**
     * Configure the order book
     *
     * @param symbol is the symbol
     * @param tobMid is the mid price
     * @param tobSpread is the spread
     * @param tobAmount is the total liquidity
     * @param depth is the depth of the order book
     */
    @Override
    public void configureOrderBook(
        String symbol, double tobMid, double tobSpread, double tobAmount, long depth
    ) {
        // apply price adjustment if present
        tobMid += priceAdjuster.getMidAdjustment(symbol);
        tobSpread += priceAdjuster.getSpreadAdjustment(symbol);

        log.debug(
            "Configuring order book for exchange {} symbol {} with mid {} spread {} amount {} depth {}",
            name,
            symbol,
            tobMid,
            tobSpread,
            tobAmount,
            depth
        );

        exchangeOrderBookContainer.updateSymbolOrderBook(symbol, tobMid, tobSpread, tobAmount, depth);
    }

    /**
     * Adjust the mid price
     *
     * @param symbol is the symbol
     * @param adjustment is the adjustment
     */
    @Override
    public void adjustMid(String symbol, double adjustment) {
        log.info("Adjusting mid for {} with {}", symbol, adjustment);
        priceAdjuster.setMidAdjustment(symbol, adjustment);
    }

    /**
     * Adjust the skew
     *
     * @param symbol is the symbol
     * @param adjustment is the adjustment
     */
    @Override
    public void adjustSkew(String symbol, double adjustment) {
        log.info("Adjusting skew for {} with {}", symbol, adjustment);
        priceAdjuster.setSpreadAdjustment(symbol, adjustment);
    }

    /**
     * Get the liquidity at a price
     *
     * @param symbol is the symbol
     * @param side is the side
     * @param price is the price
     * @return the liquidity at the price
     */
    @Override
    public double getLiquidityAtPrice(String symbol, ExchangeSide side, double price) {
        return exchangeOrderBookContainer.getLiquidityAtPrice(symbol, side, price);
    }

    /**
     * Get the filled price
     *
     * @param symbol is the symbol
     * @param side is the side
     * @param amount is the amount
     * @return the filled price
     */
    @Override
    public double getFilledPrice(String symbol, ExchangeSide side, double amount) {
        return exchangeOrderBookContainer.getFilledPrice(symbol, side, amount);
    }

    /**
     * Get the top of book price
     *
     * @param symbol is the symbol
     * @param side is the side
     * @return the top of book price
     */
    @Override
    public double getTobPrice(String symbol, ExchangeSide side) {
        return exchangeOrderBookContainer.getTobPrice(symbol, side);
    }

    /**
     * Set and start the resting orders timer
     *
     * @param restingOrdersTimer is the timer
     */
    @Override
    public void setAndStartRestingOrdersTimer(Timer restingOrdersTimer) {
        this.restingOrdersTimer = restingOrdersTimer;
        this.restingOrdersTimer.scheduleAtFixedRate(
            DEFAULT_RESTING_ORDER_RATE, DEFAULT_RESTING_ORDER_TIME_UNIT
        );
    }

    /**
     * Set and start the market data publisher timer
     *
     * @param marketDataPublisherTimer is the timer
     */
    @Override
    public void setAndStartMarketDataPublisherTimer(Timer marketDataPublisherTimer) {
        this.marketDataPublisherTimer = marketDataPublisherTimer;
        this.marketDataPublisherTimer.scheduleAtFixedRate(
            DEFAULT_MARKET_DATA_PUBLISHER_RATE, DEFAULT_MARKET_DATA_PUBLISHER_TIME_UNIT
        );
    }

    /**
     * Set and start the delayed orders timer
     *
     * @param delayedOrdersTimer is the timer
     */
    @Override
    public void setAndStartDelayedOrdersTimer(Timer delayedOrdersTimer) {
        this.delayedOrdersTimer = delayedOrdersTimer;
        this.delayedOrdersTimer.scheduleAtFixedRate(
            DEFAULT_DELAYED_ORDER_RATE, DEFAULT_DELAYED_ORDER_TIME_UNIT
        );
    }

    /**
     * Identify matching orders
     *
     * @param symbol is the symbol
     * @param side is the side
     * @param price is the price
     * @return the list of matching orders
     */
    @Override
    public List<String> identifyMatchingOrders(String symbol, Side side, double price) {
        log.info("Identifying matching orders for {} {} {}", symbol, side, price);
        return liveOrders.identifyMatchingOrders(symbol, side, price);
    }

    private String createExchangeOrderId() {
        return "A" + UUID.randomUUID().toString().substring(0, 7);
    }

    public ExchangeOrderTracker getLiveOrders() {
        return liveOrders;
    }

    public ExchangeResponse getExchangeResponse() {
        return exchangeResponse;
    }

    public ExchangeOrderBookContainer getExchangeOrderBookContainer() {
        return exchangeOrderBookContainer;
    }

    public ExchangeName getName() {
        return name;
    }

    public PriceAdjuster getPriceAdjuster() {
        return priceAdjuster;
    }

    public Timer getRestingOrdersTimer() {
        return restingOrdersTimer;
    }

    public void setRestingOrdersTimer(Timer restingOrdersTimer) {
        this.restingOrdersTimer = restingOrdersTimer;
    }

    @Override
    public OrderAction getOrderAction() {
        return this.orderAction;
    }

    @Override
    public void setOrderAction(OrderAction orderAction) {
        this.orderAction = orderAction;
    }
}



```
