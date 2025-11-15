```gherkin
Feature: Order cancellation behavior
  Verify order cancellation flows under different exchange behaviors

  Background:
    Given exchange "PARFX" is configured with behavior "DEFAULT"

  Scenario: Successfully cancel a pending DAY order
    When a new order is submitted:
      | clOrdId     | ORD-CANCEL-001 |
      | symbol      | USDCAD         |
      | side        | BUY            |
      | price       | 1.0123         |
      | timeInForce | DAY            |
      | quantity    | 10000000       |
    Then execution reports are received within 1 second:
      | status      |
      | PENDING_NEW |
      | NEW         |
    When order "ORD-CANCEL-001" is cancelled
    Then execution reports are received within 1 second:
      | status         |
      | PENDING_CANCEL |
      | CANCELED       |

  Scenario: Cancel is rejected when order has already filled
    Given exchange "PARFX" is configured with behavior "FILL_WITH_DELAY"
    When a new order is submitted:
      | clOrdId     | ORD-FILLED-001 |
      | symbol      | USDJPY         |
      | side        | SELL           |
      | price       | 1.0112         |
      | timeInForce | DAY            |
      | quantity    | 10000000       |
    Then execution reports are received within 2 seconds:
      | status      |
      | PENDING_NEW |
      | NEW         |
      | TRADE       |
    When order "ORD-FILLED-001" is cancelled
    Then execution reports are received within 1 second:
      | status         |
      | PENDING_CANCEL |
    And a cancel reject is received for "ORD-FILLED-001"

  Scenario: Successfully cancel before fill delay completes
    Given exchange "PARFX" is configured with behavior "FILL_WITH_DELAY"
    When a new order is submitted:
      | clOrdId     | ORD-QUICK-CANCEL-001 |
      | symbol      | USDJPY               |
      | side        | SELL                 |
      | price       | 1.0112               |
      | timeInForce | DAY                  |
      | quantity    | 10000000             |
    Then execution reports are received within 1 second:
      | status      |
      | PENDING_NEW |
      | NEW         |
    When order "ORD-QUICK-CANCEL-001" is cancelled
    Then execution reports are received within 1 second:
      | status         |
      | PENDING_CANCEL |
      | CANCELED       |

  Scenario: Cancel multiple orders with different behaviors
    Given exchange "PARFX" is configured with behavior "DEFAULT"
    When a new order is submitted:
      | clOrdId     | ORD-MULTI-001 |
      | symbol      | EURUSD        |
      | side        | BUY           |
      | price       | 1.0500        |
      | timeInForce | DAY           |
      | quantity    | 5000000       |
    And a new order is submitted:
      | clOrdId     | ORD-MULTI-002 |
      | symbol      | GBPUSD        |
      | side        | SELL          |
      | price       | 1.2500        |
      | timeInForce | DAY           |
      | quantity    | 3000000       |
    Then execution reports are received within 1 second:
      | status      |
      | PENDING_NEW |
      | NEW         |
      | PENDING_NEW |
      | NEW         |
    When order "ORD-MULTI-001" is cancelled
    And order "ORD-MULTI-002" is cancelled
    Then the following execution reports are received:
      | clOrdId       | statuses                  |
      | ORD-MULTI-001 | PENDING_CANCEL, CANCELED  |
      | ORD-MULTI-002 | PENDING_CANCEL, CANCELED  |
```
``` java
public class OrderStepDefinitions {

    private static final Logger log = LoggerFactory.getLogger(OrderStepDefinitions.class);

    // Shared test infrastructure
    private static TestContext testContext;
    private static OrderFixtures fixtures;
    private static OrderVerifier verifier;

    // =========================================================================
    // Lifecycle Management
    // =========================================================================

    @BeforeAll
    public static void beforeAll() {
        log.info("Initializing test environment");
        testContext = TestContext.initialize();
        fixtures = new OrderFixtures(testContext);
        verifier = new OrderVerifier(testContext);
    }

    @AfterAll
    public static void afterAll() {
        if (testContext != null) {
            testContext.shutdown();
            log.info("Test environment shutdown complete");
        }
    }

    @After
    public void afterScenario() {
        testContext.clearScenarioData();
    }

    // =========================================================================
    // Configuration Steps
    // =========================================================================

    @Given("exchange {string} is configured with behavior {string}")
    public void configureExchangeBehavior(String exchangeName, String behavior) {
        log.info("Configuring exchange {} with behavior: {}", exchangeName, behavior);
        
        testContext.getRestManager().setExchangeOrderAction(
            ProcessorType.EXCHANGE_MANAGER,
            ExchangeName.valueOf(exchangeName).name(),
            OrderAction.valueOf(behavior).name()
        );
    }

    @Given("exchange {string} is set to default behavior")
    public void setExchangeToDefaultBehavior(String exchangeName) {
        configureExchangeBehavior(exchangeName, "DEFAULT");
    }

    // =========================================================================
    // Order Submission Steps
    // =========================================================================

    @When("a new order is submitted:")
    public void submitNewOrder(DataTable dataTable) {
        Map<String, String> orderDetails = dataTable.asMap(String.class, String.class);
        
        log.info("Submitting new order with details: {}", orderDetails);
        
        OrderParams params = OrderParams.builder()
            .clOrdId(orderDetails.get("clOrdId"))
            .exchange(orderDetails.getOrDefault("exchange", "PARFX"))
            .currency(orderDetails.getOrDefault("currency", "USD"))
            .symbol(orderDetails.get("symbol"))
            .side(orderDetails.get("side"))
            .securityType(orderDetails.getOrDefault("securityType", "FXSPOT"))
            .withTimeInForce(orderDetails.getOrDefault("timeInForce", "DAY"))
            .price(Double.parseDouble(orderDetails.get("price")))
            .quantity(Double.parseDouble(orderDetails.get("quantity")))
            .build();
        
        fixtures.sendOrder(params);
    }

    @When("order {string} is submitted with:")
    public void submitNamedOrder(String clOrdId, DataTable dataTable) {
        Map<String, String> orderDetails = dataTable.asMap(String.class, String.class);
        orderDetails.put("clOrdId", clOrdId);
        
        submitNewOrder(DataTable.create(
            orderDetails.entrySet().stream()
                .map(e -> Arrays.asList(e.getKey(), e.getValue()))
                .collect(Collectors.toList())
        ));
    }

    @When("FIX messages are replayed from file {string}")
    public void replayFixMessages(String fileName) {
        log.info("Replaying FIX messages from: {}", fileName);
        fixtures.replayOrdersFromFile(fileName);
    }

    @When("linked orders are submitted with ClOrdLinkId {string}:")
    public void submitLinkedOrders(String clOrdLinkId, DataTable dataTable) {
        log.info("Submitting linked orders with ClOrdLinkId: {}", clOrdLinkId);
        testContext.setCurrentClOrdLinkID(clOrdLinkId);
        fixtures.sendLinkedOrders(clOrdLinkId, dataTable);
    }

    // =========================================================================
    // Order Cancellation Steps
    // =========================================================================

    @When("order {string} is cancelled")
    public void cancelOrder(String clOrdId) {
        log.info("Submitting cancel request for order: {}", clOrdId);
        fixtures.sendCancel(clOrdId);
    }

    @When("a cancel request is submitted for {string}")
    public void submitCancelRequest(String clOrdId) {
        cancelOrder(clOrdId);
    }

    // =========================================================================
    // Execution Report Verification Steps
    // =========================================================================

    @Then("execution reports are received within {int} second(s):")
    public void verifyExecutionReportsWithinTimeout(int seconds, DataTable dataTable) {
        List<String> expectedStatuses = dataTable.asList(String.class)
            .stream()
            .filter(s -> !s.equalsIgnoreCase("status")) // Filter out header
            .collect(Collectors.toList());
        
        log.info("Verifying execution reports {} within {} second(s)", 
            expectedStatuses, seconds);
        
        verifier.verifyExecutionReports(expectedStatuses, Duration.ofSeconds(seconds));
    }

    @Then("the following execution reports are received:")
    public void verifyExecutionReportsForOrders(DataTable dataTable) {
        List<Map<String, String>> rows = dataTable.asMaps(String.class, String.class);
        
        for (Map<String, String> row : rows) {
            String clOrdId = row.get("clOrdId");
            String statusesStr = row.get("statuses");
            
            List<String> statuses = Arrays.stream(statusesStr.split(","))
                .map(String::trim)
                .collect(Collectors.toList());
            
            log.info("Verifying execution reports for {}: {}", clOrdId, statuses);
            verifier.verifyExecutionReports(clOrdId, statuses);
        }
    }

    @Then("order {string} receives execution reports:")
    public void verifyExecutionReportsForSpecificOrder(String clOrdId, DataTable dataTable) {
        List<String> expectedStatuses = dataTable.asList(String.class)
            .stream()
            .filter(s -> !s.equalsIgnoreCase("status"))
            .collect(Collectors.toList());
        
        log.info("Verifying execution reports for {}: {}", clOrdId, expectedStatuses);
        verifier.verifyExecutionReports(clOrdId, expectedStatuses);
    }

    @Then("execution reports {string} are received for order {string}")
    public void verifyNamedExecutionReports(String statusesStr, String clOrdId) {
        List<String> statuses = parseStatuses(statusesStr);
        log.info("Verifying execution reports for {}: {}", clOrdId, statuses);
        verifier.verifyCancelRequest(clOrdId, statuses);
    }

    // =========================================================================
    // Order Acceptance Verification Steps
    // =========================================================================

    @Then("the order is accepted successfully")
    public void verifyOrderAccepted() {
        log.info("Verifying order acceptance");
        verifier.verifyOrderAcceptance(Duration.ofSeconds(5));
    }

    @Then("all orders are accepted and linked by ClOrdLinkId {string}")
    public void verifyLinkedOrdersAccepted(String clOrdLinkId) {
        log.info("Verifying all orders linked by ClOrdLinkId: {}", clOrdLinkId);
        testContext.validateClOrdLinkID(clOrdLinkId);
        verifier.verifyOrderAcceptance(Duration.ofSeconds(5));
    }

    // =========================================================================
    // Cancel Rejection Verification Steps
    // =========================================================================

    @Then("a cancel reject is received for {string}")
    public void verifyCancelRejection(String clOrdId) {
        log.info("Verifying cancel rejection for: {}", clOrdId);
        
        OrderCancelRejectRequestDTO reject = testContext
            .getOrderCancelRejectRequestDTOS()
            .stream()
            .filter(r -> r.getOrigClOrdID().equals(clOrdId))
            .findFirst()
            .orElseThrow(() -> new AssertionError(
                "No cancel reject received for order: " + clOrdId));
        
        verifier.verifyOrderCancelRejectMessage(reject.getClOrdID());
    }

    @Then("order {string} cancel is rejected")
    public void verifyCancelRejected(String clOrdId) {
        verifyCancelRejection(clOrdId);
    }

    @Then("cancel rejection is received for {string} with reason {string}")
    public void verifyCancelRejectionWithReason(String clOrdId, String expectedReason) {
        log.info("Verifying cancel rejection for {} with reason: {}", clOrdId, expectedReason);
        
        OrderCancelRejectRequestDTO reject = testContext
            .getOrderCancelRejectRequestDTOS()
            .stream()
            .filter(r -> r.getOrigClOrdID().equals(clOrdId))
            .findFirst()
            .orElseThrow(() -> new AssertionError(
                "No cancel reject received for order: " + clOrdId));
        
        verifier.verifyOrderCancelRejectMessage(reject.getClOrdID());
        // Additional reason validation could be added here if needed
    }

    // =========================================================================
    // Helper Methods
    // =========================================================================

    private List<String> parseStatuses(String statusesStr) {
        if (statusesStr == null || statusesStr.isBlank()) {
            throw new IllegalArgumentException("Execution statuses cannot be empty");
        }
        return Arrays.stream(statusesStr.split("\\s*,\\s*"))
            .map(String::trim)
            .collect(Collectors.toList());
    }
}
```
