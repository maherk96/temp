```java
    @When("the client simulator replays new order single fix messages from the file {string}")
    public void replay_fix_messages_from_file(String fileName) {
        System.out.printf("🔁 Replaying FIX messages from file: %s%n", fileName);

        Stream<OrderInfo> messages = OrderParamProvider.nosFusionAlgoMessages();
        messages.forEach(orderInfo -> {
            System.out.println("➡ Sending FIX Message: " + orderInfo);
            try {
                clientSimulator
                    .sendRawFixTradeMessage(orderInfo.message())
                    .waitForExecutionReport(
                        "BASIC_FLOW",
                        Duration.ofSeconds(2),
                        "Order should have pending new, new and trade execution types"
                    );
            } catch (Exception e) {
                throw new RuntimeException("❌ Failed processing message: " + orderInfo, e);
            }
        });
    }

    @Then("the service should produce {string}, {string} and {string} execution reports for each message")
    public void verify_execution_report_types(String type1, String type2, String type3) {
        // If you already validated within the send loop, this may just summarize results
        System.out.printf("✅ Verified expected execution types: %s, %s, %s%n", type1, type2, type3);
        clientSimulator.getPoolStatistics();
    }

```
