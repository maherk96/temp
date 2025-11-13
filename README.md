```java
package com.trading.test.steps;

import com.trading.test.fixtures.TestContext;
import com.trading.test.fixtures.OrderFixtures;
import com.trading.test.verification.OrderVerifier;
import com.trading.test.builders.OrderBuilder;
import com.trading.test.data.OrderParams;
import io.cucumber.java.After;
import io.cucumber.java.BeforeAll;
import io.cucumber.java.AfterAll;
import io.cucumber.java.en.Given;
import io.cucumber.java.en.When;
import io.cucumber.java.en.Then;
import io.cucumber.datatable.DataTable;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Duration;
import java.util.List;

/**
 * Cucumber step definitions for order processing tests.
 * Coordinates test fixtures, builders, and verifiers.
 */
public class OrderStepDefinitions {

    private static final Logger log = LoggerFactory.getLogger(OrderStepDefinitions.class);

    // Shared instances across all scenarios
    private static TestContext testContext;
    private static OrderFixtures fixtures;
    private static OrderVerifier verifier;
    private static OrderBuilder orderBuilder;

    @BeforeAll
    public static void beforeAll() {
        log.info("Initializing test environment");
        testContext = TestContext.initialize();
        fixtures = new OrderFixtures(testContext);
        verifier = new OrderVerifier(testContext);
        orderBuilder = new OrderBuilder();
        log.info("Test environment initialized successfully");
    }

    @AfterAll
    public static void afterAll() {
        log.info("Shutting down test environment");
        if (testContext != null) {
            testContext.shutdown();
        }
        log.info("Test environment shut down successfully");
    }

    @After
    public void afterScenario() {
        log.debug("Cleaning up scenario data");
        testContext.clearScenarioData();
    }

    @Given("a unique ClOrdLinkId {string} is generated")
    public void aUniqueClOrdLinkIDIsGenerated(String clOrdLinkID) {
        testContext.setCurrentClOrdLinkID(clOrdLinkID);
        log.info("Scenario-specific Client Order Link ID set to: {}", clOrdLinkID);
    }

    @When("the client simulator replays new order single fix messages from the file {string}")
    public void replayFixMessagesFromFile(String fileName) {
        log.info("Replaying FIX messages from file: {}", fileName);
        fixtures.replayOrdersFromFile(fileName);
    }

    @When("the client sends a new order to an exchange {string} with currency {string}, symbol {string}, side {string}, security type {string}, price {string}, quantity {string}")
    public void sendNewOrder(String exchange, String currency, String symbol, 
                           String side, String securityType, String price, String quantity) {
        log.info("Creating new order: exchange={}, currency={}, symbol={}, side={}, securityType={}, price={}, quantity={}",
                exchange, currency, symbol, side, securityType, price, quantity);

        OrderParams params = OrderParams.builder()
                .exchange(exchange)
                .currency(currency)
                .symbol(symbol)
                .side(side)
                .securityType(securityType)
                .price(Double.parseDouble(price))
                .quantity(Double.parseDouble(quantity))
                .build();

        fixtures.sendOrder(params);
    }

    @When("the client places a new order single with time in force value set to {string}")
    public void placeOrderWithTimeInForce(String timeInForce) {
        log.info("Creating new order single with time in force: {}", timeInForce);

        OrderParams params = OrderParams.createDefault()
                .withTimeInForce(timeInForce);

        fixtures.sendOrder(params);
    }

    @When("the client places a new order single with security type {string}, symbol {string} and time in force value set to {string}")
    public void placeOrderWithSecurityTypeAndTimeInForce(String securityType, String symbol, String timeInForce) {
        log.info("Creating new order: securityType={}, symbol={}, timeInForce={}", 
                securityType, symbol, timeInForce);

        OrderParams params = OrderParams.createDefault()
                .withSecurityType(securityType)
                .withSymbol(symbol)
                .withCurrency(symbol.substring(0, 3))
                .withTimeInForce(timeInForce);

        fixtures.sendOrder(params);
    }

    @When("the client sends the following orders linked by {string}:")
    public void sendLinkedOrders(String clOrdLinkID, DataTable dataTable) {
        testContext.validateClOrdLinkID(clOrdLinkID);
        
        log.info("Sending linked orders for ClOrdLinkID: {}", clOrdLinkID);
        fixtures.sendLinkedOrders(clOrdLinkID, dataTable);
    }

    @Then("the service should produce {string} execution reports for each message within {int} seconds")
    public void verifyExecutionReportTypes(String executionTypes, int seconds) {
        List<String> execTypes = parseExecutionTypes(executionTypes);
        log.info("Verifying execution report types: {} within {} seconds", execTypes, seconds);
        
        verifier.verifyExecutionReports(execTypes, Duration.ofSeconds(seconds));
    }

    @Then("the order should be accepted successfully")
    public void theOrderShouldBeAcceptedSuccessfully() {
        log.info("Verifying order acceptance");
        verifier.verifyOrderAcceptance(Duration.ofSeconds(5));
    }

    @Then("all orders should be accepted successfully and linked by clOrdLinkId {string}")
    public void allOrdersShouldBeAcceptedAndLinked(String expectedClOrdLinkID) {
        testContext.validateClOrdLinkID(expectedClOrdLinkID);
        
        log.info("Verifying all orders linked by ClOrdLinkID: {}", expectedClOrdLinkID);
        verifier.verifyOrderAcceptance(Duration.ofSeconds(5));
        log.info("All orders for ClOrdLinkID '{}' accepted successfully", expectedClOrdLinkID);
    }

    private List<String> parseExecutionTypes(String executionTypes) {
        if (executionTypes == null || executionTypes.isBlank()) {
            throw new IllegalArgumentException("Execution types cannot be empty");
        }
        return List.of(executionTypes.split("\\s*,\\s*"));
    }
}

package com.trading.test.fixtures;

import com.trading.core.environment.TemporaryEnvironment;
import com.trading.core.services.ServiceState;
import com.trading.exchange.ExchangeManager;
import com.trading.fixgateway.FixGateway;
import com.trading.orderprocessor.OrderProcessor;
import com.trading.test.simulator.CORMockFixClientSimulator;
import net.openhft.chronicle.core.Jvm;
import net.openhft.chronicle.core.time.Clocks;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.concurrent.TimeUnit;
import java.util.function.BooleanSupplier;

/**
 * Manages test context including services, simulators, and scenario state.
 * Singleton pattern ensures single instance across all test scenarios.
 */
public class TestContext {

    private static final Logger log = LoggerFactory.getLogger(TestContext.class);
    private static final long SERVICE_STARTUP_TIMEOUT_SECONDS = 10;
    
    private static TestContext instance;

    private final TemporaryEnvironment env;
    private final OrderProcessor orderProcessor;
    private final FixGateway fixGateway;
    private final ExchangeManager exchangeManager;
    private final CORMockFixClientSimulator clientSimulator;

    // Scenario-specific state (cleared between scenarios)
    private final List<String> clOrderIdList;
    private String currentClOrdLinkID;

    private TestContext() {
        log.info("Initializing test context");
        
        this.env = new TemporaryEnvironment();
        this.clOrderIdList = new ArrayList<>();
        
        try {
            var config = env.loadConfig("services.yaml");

            // Start services
            this.orderProcessor = startService(config, "order-processor", OrderProcessor.class);
            this.fixGateway = startService(config, "fix-gateway", FixGateway.class);
            this.exchangeManager = startService(config, "exchange-manager", ExchangeManager.class);

            // Wait for services to become active
            waitForServiceActive(orderProcessor::getState, "OrderProcessor");
            waitForServiceActive(fixGateway::getState, "FixGateway");
            waitForServiceActive(exchangeManager::getState, "ExchangeManager");

            // Initialize client simulator
            this.clientSimulator = new CORMockFixClientSimulator(Set.of("fusion_local_1"));
            
            log.info("Test context initialized successfully");
            
        } catch (IOException e) {
            throw new TestEnvironmentException("Failed to initialize test context", e);
        }
    }

    public static synchronized TestContext initialize() {
        if (instance == null) {
            instance = new TestContext();
        }
        return instance;
    }

    public static TestContext getInstance() {
        if (instance == null) {
            throw new IllegalStateException("TestContext not initialized. Call initialize() first.");
        }
        return instance;
    }

    @SuppressWarnings("unchecked")
    private <T> T startService(Object config, String serviceName, Class<T> serviceClass) {
        try {
            log.info("Starting service: {}", serviceName);
            var service = env.getRunner()
                    .start(((ConfigWrapper) config).getServiceByName(serviceName))
                    .getImplementation();
            return (T) service;
        } catch (Exception e) {
            throw new TestEnvironmentException("Failed to start service: " + serviceName, e);
        }
    }

    private void waitForServiceActive(java.util.function.Supplier<ServiceState> stateSupplier, String serviceName) {
        log.info("Waiting for {} to become active", serviceName);
        waitForCondition(
                () -> stateSupplier.get() == ServiceState.ACTIVE,
                SERVICE_STARTUP_TIMEOUT_SECONDS,
                serviceName + " failed to become active"
        );
        log.info("{} is now active", serviceName);
    }

    private void waitForCondition(BooleanSupplier supplier, long timeoutSeconds, String errorMessage) {
        var timeout = Clocks.defaultClock().getEpochNanos() + TimeUnit.SECONDS.toNanos(timeoutSeconds);
        while (Clocks.defaultClock().getEpochNanos() < timeout) {
            if (supplier.getAsBoolean()) {
                return;
            }
            Jvm.pause(50);
        }
        throw new TestEnvironmentException(errorMessage + " (timeout after " + timeoutSeconds + "s)");
    }

    public void shutdown() {
        log.info("Shutting down test context");
        try {
            if (clientSimulator != null) {
                clientSimulator.close();
            }
            if (env != null) {
                env.close();
            }
            log.info("Test context shut down successfully");
        } catch (IOException e) {
            log.error("Error during test context shutdown", e);
        }
    }

    // Scenario state management
    public void clearScenarioData() {
        clOrderIdList.clear();
        currentClOrdLinkID = null;
    }

    public void addOrderId(String orderId) {
        clOrderIdList.add(orderId);
    }

    public List<String> getTrackedOrderIds() {
        return new ArrayList<>(clOrderIdList);
    }

    public void setCurrentClOrdLinkID(String clOrdLinkID) {
        this.currentClOrdLinkID = clOrdLinkID;
    }

    public String getCurrentClOrdLinkID() {
        return currentClOrdLinkID;
    }

    public void validateClOrdLinkID(String expectedClOrdLinkID) {
        if (currentClOrdLinkID == null || !currentClOrdLinkID.equals(expectedClOrdLinkID)) {
            throw new AssertionError(
                    String.format("Expected ClOrdLinkID '%s' but current is '%s'",
                            expectedClOrdLinkID, currentClOrdLinkID));
        }
    }

    // Getters
    public OrderProcessor getOrderProcessor() {
        return orderProcessor;
    }

    public FixGateway getFixGateway() {
        return fixGateway;
    }

    public ExchangeManager getExchangeManager() {
        return exchangeManager;
    }

    public CORMockFixClientSimulator getClientSimulator() {
        return clientSimulator;
    }

    // Placeholder interface for config
    private interface ConfigWrapper {
        Object getServiceByName(String name);
    }

    public static class TestEnvironmentException extends RuntimeException {
        public TestEnvironmentException(String message) {
            super(message);
        }

        public TestEnvironmentException(String message, Throwable cause) {
            super(message, cause);
        }
    }
}

package com.trading.test.fixtures;

import com.trading.dto.NewOrderSingleDTO;
import com.trading.test.data.OrderParams;
import com.trading.test.data.TestRequestDataHelper;
import com.trading.test.providers.OrderParamProvider;
import com.trading.test.providers.StreamOrderParamProvider;
import io.cucumber.datatable.DataTable;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import quickfix.field.ClOrdID;

/**
 * Handles order creation and submission to the test environment.
 * Manages interaction with client simulator and order tracking.
 */
public class OrderFixtures {

    private static final Logger log = LoggerFactory.getLogger(OrderFixtures.class);

    private final TestContext testContext;
    private final TestRequestDataHelper dataHelper;

    public OrderFixtures(TestContext testContext) {
        this.testContext = testContext;
        this.dataHelper = new TestRequestDataHelper();
    }

    /**
     * Replays FIX messages from a file and tracks all order IDs.
     */
    public void replayOrdersFromFile(String fileName) {
        log.info("Replaying FIX messages from file: {}", fileName);

        StreamOrderParamProvider.OrderInfo<?> messages =
                OrderParamProvider.noosFusionAlgoMessages(fileName);

        messages.forEach(orderInfo -> {
            try {
                testContext.getClientSimulator().sendRawFixTradeMessage(orderInfo.message());
                String clorderId = orderInfo.message().getString(new ClOrdID().getTag());
                testContext.addOrderId(clorderId);
                log.debug("Sent order from file: {}", clorderId);
            } catch (Exception e) {
                log.error("Error processing message: {}", orderInfo, e);
                throw new OrderProcessingException("Failed to process message from file: " + fileName, e);
            }
        });

        log.info("Successfully replayed {} orders from file: {}", 
                messages.getOrderCount(), fileName);
    }

    /**
     * Sends a single order based on provided parameters.
     */
    public void sendOrder(OrderParams params) {
        try {
            NewOrderSingleDTO order = createOrder(params);
            sendOrderInternal(order);
        } catch (Exception e) {
            log.error("Failed to send order: {}", params, e);
            throw new OrderProcessingException("Failed to send order", e);
        }
    }

    /**
     * Sends multiple linked orders from a DataTable.
     */
    public void sendLinkedOrders(String clOrdLinkID, DataTable dataTable) {
        dataTable.asMaps(String.class, String.class).forEach(row -> {
            try {
                OrderParams params = OrderParams.fromDataTableRow(row);
                NewOrderSingleDTO order = createOrder(params);
                
                // Override with linked order details
                String childOrderId = row.get("childOrderId");
                if (childOrderId != null && !childOrderId.isBlank()) {
                    order.setClOrdID(childOrderId);
                }
                order.setClOrdLinkID(clOrdLinkID);

                log.info("Sending linked order: ClOrdID={}, ClOrdLinkID={}, Exchange={}, Symbol={}, Price={}, Quantity={}",
                        order.getClOrdID(), order.getClOrdLinkID(), 
                        order.getCustOriginalExecAuthority(), order.getSymbol(),
                        order.getPrice(), order.getOrderQty());

                sendOrderInternal(order);
            } catch (Exception e) {
                log.error("Failed to send linked order from row: {}", row, e);
                throw new OrderProcessingException("Failed to send linked order", e);
            }
        });
    }

    private NewOrderSingleDTO createOrder(OrderParams params) {
        return dataHelper.createNewOrderSingle(
                params.getExchange(),
                params.getCurrency(),
                params.getSymbol(),
                params.getSide(),
                params.getSecurityType(),
                params.getPrice(),
                params.getQuantity()
        );
    }

    private void sendOrderInternal(NewOrderSingleDTO order) {
        // Apply time in force if specified
        if (order.getTimeInForce() == null) {
            // Set default if not already set
            order.setTimeInForce(TimeInForce.DAY);
        }

        testContext.getClientSimulator().sendTradeMessage(order);
        testContext.addOrderId(order.getClOrdID());
        log.debug("Sent order: ClOrdID={}", order.getClOrdID());
    }

    public static class OrderProcessingException extends RuntimeException {
        public OrderProcessingException(String message, Throwable cause) {
            super(message, cause);
        }
    }
}

package com.trading.test.verification;

import com.trading.dto.NewOrderSingleDTO;
import com.trading.enums.ExecutionType;
import com.trading.test.fixtures.TestContext;
import com.trading.test.wait.CORClientSimWaitFactory;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Duration;
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

/**
 * Handles verification of order execution reports and state.
 * Provides reusable assertion methods for different verification scenarios.
 */
public class OrderVerifier {

    private static final Logger log = LoggerFactory.getLogger(OrderVerifier.class);
    
    private static final List<ExecutionType> DEFAULT_ACCEPTANCE_TYPES = 
            List.of(ExecutionType.PENDING_NEW, ExecutionType.NEW);

    private final TestContext testContext;

    public OrderVerifier(TestContext testContext) {
        this.testContext = testContext;
    }

    /**
     * Verifies that all tracked orders have the expected execution report types.
     */
    public void verifyExecutionReports(List<String> executionTypeNames, Duration timeout) {
        List<ExecutionType> expectedTypes = parseExecutionTypes(executionTypeNames);
        
        log.info("Verifying {} execution reports of types: {} within {}",
                testContext.getTrackedOrderIds().size(), expectedTypes, timeout);

        verifyOrdersHaveExecutionTypes(
                testContext.getTrackedOrderIds(),
                expectedTypes,
                timeout,
                "Order should have execution types: " + expectedTypes
        );
    }

    /**
     * Verifies that all tracked orders are accepted (PENDING_NEW, NEW).
     */
    public void verifyOrderAcceptance(Duration timeout) {
        log.info("Verifying {} orders are accepted within {}",
                testContext.getTrackedOrderIds().size(), timeout);

        verifyOrdersHaveExecutionTypes(
                testContext.getTrackedOrderIds(),
                DEFAULT_ACCEPTANCE_TYPES,
                timeout,
                "Order should be accepted (PENDING_NEW, NEW)"
        );
    }

    /**
     * Core verification method that checks execution reports for given orders.
     */
    private void verifyOrdersHaveExecutionTypes(
            List<String> orderIds,
            List<ExecutionType> expectedTypes,
            Duration timeout,
            String assertionMessage) {

        if (orderIds.isEmpty()) {
            log.warn("No orders to verify");
            return;
        }

        int successCount = 0;
        int failureCount = 0;

        for (String orderId : orderIds) {
            try {
                log.debug("Verifying order: {}", orderId);
                
                new CORClientSimWaitFactory(
                        testContext.getClientSimulator().getCorClientSimFixMessageQueue(),
                        NewOrderSingleDTO.builder().clordID(orderId).build()
                )
                .waitForExecutionReport(expectedTypes, timeout, assertionMessage);
                
                successCount++;
                log.debug("Order {} verified successfully", orderId);
                
            } catch (Exception e) {
                failureCount++;
                log.error("Failed to verify order {}: {}", orderId, e.getMessage(), e);
                throw new OrderVerificationException(
                        "Order verification failed for " + orderId, e);
            }
        }

        log.info("Verification complete: {} succeeded, {} failed", successCount, failureCount);
    }

    private List<ExecutionType> parseExecutionTypes(List<String> typeNames) {
        try {
            return typeNames.stream()
                    .map(String::trim)
                    .map(ExecutionType::valueOf)
                    .collect(Collectors.toList());
        } catch (IllegalArgumentException e) {
            throw new IllegalArgumentException(
                    "Invalid execution type. Valid values: " + 
                    Arrays.toString(ExecutionType.values()), e);
        }
    }

    public static class OrderVerificationException extends RuntimeException {
        public OrderVerificationException(String message, Throwable cause) {
            super(message, cause);
        }
    }
}

package com.trading.test.data;

import com.trading.enums.ExchangeName;
import com.trading.enums.SecurityType;
import com.trading.enums.Side;
import com.trading.enums.TimeInForce;

import java.util.Map;

/**
 * Immutable parameter object for order creation.
 * Provides builder pattern and factory methods for common scenarios.
 */
public class OrderParams {

    // Default values for common test scenarios
    private static final String DEFAULT_EXCHANGE = "PARFX";
    private static final String DEFAULT_CURRENCY = "EUR";
    private static final String DEFAULT_SYMBOL = "EURUSD";
    private static final String DEFAULT_SIDE = "BUY";
    private static final String DEFAULT_SECURITY_TYPE = "FXSPOT";
    private static final double DEFAULT_PRICE = 1.312;
    private static final double DEFAULT_QUANTITY = 1_000_000;

    private final ExchangeName exchange;
    private final String currency;
    private final String symbol;
    private final Side side;
    private final SecurityType securityType;
    private final double price;
    private final double quantity;
    private final TimeInForce timeInForce;

    private OrderParams(Builder builder) {
        this.exchange = ExchangeName.valueOf(builder.exchange);
        this.currency = builder.currency;
        this.symbol = builder.symbol;
        this.side = Side.valueOf(builder.side);
        this.securityType = SecurityType.valueOf(builder.securityType);
        this.price = builder.price;
        this.quantity = builder.quantity;
        this.timeInForce = builder.timeInForce != null ? 
                TimeInForce.valueOf(builder.timeInForce) : null;
    }

    public static Builder builder() {
        return new Builder();
    }

    /**
     * Creates default PARFX EUR/USD order parameters.
     */
    public static Builder createDefault() {
        return builder()
                .exchange(DEFAULT_EXCHANGE)
                .currency(DEFAULT_CURRENCY)
                .symbol(DEFAULT_SYMBOL)
                .side(DEFAULT_SIDE)
                .securityType(DEFAULT_SECURITY_TYPE)
                .price(DEFAULT_PRICE)
                .quantity(DEFAULT_QUANTITY);
    }

    /**
     * Creates OrderParams from a Cucumber DataTable row.
     */
    public static OrderParams fromDataTableRow(Map<String, String> row) {
        return builder()
                .exchange(row.get("exchange"))
                .currency(row.get("currency"))
                .symbol(row.get("symbol"))
                .side(row.get("side"))
                .securityType(row.get("securityType"))
                .price(Double.parseDouble(row.get("price")))
                .quantity(Double.parseDouble(row.get("quantity")))
                .build();
    }

    // Getters
    public ExchangeName getExchange() {
        return exchange;
    }

    public String getCurrency() {
        return currency;
    }

    public String getSymbol() {
        return symbol;
    }

    public Side getSide() {
        return side;
    }

    public SecurityType getSecurityType() {
        return securityType;
    }

    public double getPrice() {
        return price;
    }

    public double getQuantity() {
        return quantity;
    }

    public TimeInForce getTimeInForce() {
        return timeInForce;
    }

    @Override
    public String toString() {
        return "OrderParams{" +
                "exchange=" + exchange +
                ", currency='" + currency + '\'' +
                ", symbol='" + symbol + '\'' +
                ", side=" + side +
                ", securityType=" + securityType +
                ", price=" + price +
                ", quantity=" + quantity +
                ", timeInForce=" + timeInForce +
                '}';
    }

    public static class Builder {
        private String exchange = DEFAULT_EXCHANGE;
        private String currency = DEFAULT_CURRENCY;
        private String symbol = DEFAULT_SYMBOL;
        private String side = DEFAULT_SIDE;
        private String securityType = DEFAULT_SECURITY_TYPE;
        private double price = DEFAULT_PRICE;
        private double quantity = DEFAULT_QUANTITY;
        private String timeInForce;

        public Builder exchange(String exchange) {
            this.exchange = exchange;
            return this;
        }

        public Builder currency(String currency) {
            this.currency = currency;
            return this;
        }

        public Builder symbol(String symbol) {
            this.symbol = symbol;
            return this;
        }

        public Builder side(String side) {
            this.side = side;
            return this;
        }

        public Builder securityType(String securityType) {
            this.securityType = securityType;
            return this;
        }

        public Builder price(double price) {
            this.price = price;
            return this;
        }

        public Builder quantity(double quantity) {
            this.quantity = quantity;
            return this;
        }

        public Builder withTimeInForce(String timeInForce) {
            this.timeInForce = timeInForce;
            return this;
        }

        public Builder withSecurityType(String securityType) {
            this.securityType = securityType;
            return this;
        }

        public Builder withSymbol(String symbol) {
            this.symbol = symbol;
            return this;
        }

        public Builder withCurrency(String currency) {
            this.currency = currency;
            return this;
        }

        public OrderParams build() {
            validateRequiredFields();
            return new OrderParams(this);
        }

        private void validateRequiredFields() {
            if (exchange == null || exchange.isBlank()) {
                throw new IllegalStateException("Exchange is required");
            }
            if (currency == null || currency.isBlank()) {
                throw new IllegalStateException("Currency is required");
            }
            if (symbol == null || symbol.isBlank()) {
                throw new IllegalStateException("Symbol is required");
            }
            if (price <= 0) {
                throw new IllegalStateException("Price must be positive");
            }
            if (quantity <= 0) {
                throw new IllegalStateException("Quantity must be positive");
            }
        }
    }
}

package com.trading.test.builders;

import com.trading.dto.NewOrderSingleDTO;
import com.trading.enums.ExchangeName;
import com.trading.enums.SecurityType;
import com.trading.enums.Side;
import com.trading.enums.TimeInForce;

import java.util.UUID;

/**
 * Fluent builder for creating NewOrderSingleDTO test objects.
 * Provides sensible defaults and validation.
 */
public class OrderBuilder {

    private ExchangeName exchange = ExchangeName.PARFX;
    private String currency = "EUR";
    private String symbol = "EURUSD";
    private Side side = Side.BUY;
    private SecurityType securityType = SecurityType.FXSPOT;
    private double price = 1.312;
    private double quantity = 1_000_000;
    private TimeInForce timeInForce = TimeInForce.DAY;
    private String clOrdID;
    private String clOrdLinkID;

    public OrderBuilder forExchange(ExchangeName exchange) {
        this.exchange = exchange;
        return this;
    }

    public OrderBuilder withCurrency(String currency) {
        this.currency = currency;
        return this;
    }

    public OrderBuilder withSymbol(String symbol) {
        this.symbol = symbol;
        return this;
    }

    public OrderBuilder withSide(Side side) {
        this.side = side;
        return this;
    }

    public OrderBuilder withSecurityType(SecurityType securityType) {
        this.securityType = securityType;
        return this;
    }

    public OrderBuilder withPrice(double price) {
        if (price <= 0) {
            throw new IllegalArgumentException("Price must be positive");
        }
        this.price = price;
        return this;
    }

    public OrderBuilder withQuantity(double quantity) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }
        this.quantity = quantity;
        return this;
    }

    public OrderBuilder withTimeInForce(TimeInForce timeInForce) {
        this.timeInForce = timeInForce;
        return this;
    }

    public OrderBuilder withClOrdID(String clOrdID) {
        this.clOrdID = clOrdID;
        return this;
    }

    public OrderBuilder withClOrdLinkID(String clOrdLinkID) {
        this.clOrdLinkID = clOrdLinkID;
        return this;
    }

    public OrderBuilder buyOrder() {
        this.side = Side.BUY;
        return this;
    }

    public OrderBuilder sellOrder() {
        this.side = Side.SELL;
        return this;
    }

    public OrderBuilder fxSpot() {
        this.securityType = SecurityType.FXSPOT;
        return this;
    }

    public NewOrderSingleDTO build() {
        validate();
        
        NewOrderSingleDTO order = new NewOrderSingleDTO();
        
        // Set required fields
        order.setCustOriginalExecAuthority(exchange.name());
        order.setCurrency(currency);
        order.setSymbol(symbol);
        order.setSide(side);
        order.setSecurityType(securityType);
        order.setPrice(price);
        order.setOrderQty(quantity);
        order.setTimeInForce(timeInForce);
        
        // Set client order ID (generate if not provided)
        order.setClOrdID(clOrdID != null ? clOrdID : generateClOrdID());
        
        // Set link ID if provided
        if (clOrdLinkID != null) {
            order.setClOrdLinkID(clOrdLinkID);
        }
        
        return order;
    }

    private void validate() {
        if (exchange == null) {
            throw new IllegalStateException("Exchange is required");
        }
        if (currency == null || currency.isBlank()) {
            throw new IllegalStateException("Currency is required");
        }
        if (symbol == null || symbol.isBlank()) {
            throw new IllegalStateException("Symbol is required");
        }
        if (side == null) {
            throw new IllegalStateException("Side is required");
        }
        if (securityType == null) {
            throw new IllegalStateException("Security type is required");
        }
    }

    private String generateClOrdID() {
        return "TEST-" + UUID.randomUUID().toString().substring(0, 8);
    }

    /**
     * Creates a new builder with default EUR/USD FX spot order.
     */
    public static OrderBuilder defaultEurUsd() {
        return new OrderBuilder();
    }

    /**
     * Creates a new builder with custom symbol and extracted currency.
     */
    public static OrderBuilder forSymbol(String symbol) {
        String currency = symbol.length() >= 3 ? symbol.substring(0, 3) : "EUR";
        return new OrderBuilder()
                .withSymbol(symbol)
                .withCurrency(currency);
    }

    /**
     * Resets builder to default state for reuse.
     */
    public OrderBuilder reset() {
        this.exchange = ExchangeName.PARFX;
        this.currency = "EUR";
        this.symbol = "EURUSD";
        this.side = Side.BUY;
        this.securityType = SecurityType.FXSPOT;
        this.price = 1.312;
        this.quantity = 1_000_000;
        this.timeInForce = TimeInForce.DAY;
        this.clOrdID = null;
        this.clOrdLinkID = null;
        return this;
    }
}
```
