```java
package com.citi.fx.qa.mock.cor.orders.manifest;

import com.citi.fx.qa.mock.cor.gen.messages.datamodel.DefaultNewOrderSingle;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;

import static org.junit.jupiter.api.Assertions.*;

@DisplayName("OrderManager Defensive Copy Tests")
class OrderManagerDefensiveCopyTest {

    private OrderManager orderManager;
    private static final String SESSION_ID = "FIX.4.4:TEST_SESSION";

    @BeforeEach
    void setUp() {
        orderManager = new OrderManager();
    }

    @Test
    @DisplayName("Bug: Concurrent mutations corrupt cached orders (SHOULD FAIL WITHOUT FIX)")
    void testConcurrentMutationsCorruptCache() throws InterruptedException, ExecutionException {
        int orderCount = 10;
        ExecutorService executor = Executors.newFixedThreadPool(orderCount);
        
        // Step 1: Create and store orders concurrently
        List<Future<OrderData>> futures = new ArrayList<>();
        
        for (int i = 1; i <= orderCount; i++) {
            final String orderRef = "ORDER_" + i;
            final String expectedClOrdID = orderRef;
            
            Future<OrderData> future = executor.submit(() -> {
                // Create order
                var order = createTestOrder(expectedClOrdID);
                
                // Store in OrderManager
                orderManager.addOrder(SESSION_ID, orderRef, order);
                
                // Return both the original reference and expected ID
                return new OrderData(order, orderRef, expectedClOrdID);
            });
            
            futures.add(future);
        }
        
        // Wait for all orders to be stored
        List<OrderData> orderDataList = new ArrayList<>();
        for (Future<OrderData> future : futures) {
            orderDataList.add(future.get());
        }
        
        // Small delay to ensure all orders are stored
        Thread.sleep(100);
        
        // Step 2: SIMULATE EXTERNAL MUTATIONS (like exchange processing does)
        // This simulates what happens when exchange code modifies the original order
        List<Future<Void>> mutationFutures = new ArrayList<>();
        
        for (OrderData orderData : orderDataList) {
            Future<Void> future = executor.submit(() -> {
                // MUTATE the original order object (simulating exchange processing)
                orderData.originalOrder.setClOrdID("MUTATED_" + orderData.originalClOrdID);
                return null;
            });
            mutationFutures.add(future);
        }
        
        // Wait for all mutations to complete
        for (Future<Void> future : mutationFutures) {
            future.get();
        }
        
        // Small delay to ensure mutations are visible
        Thread.sleep(100);
        
        // Step 3: VERIFY - Retrieved orders should NOT be mutated
        // This is where the bug manifests!
        List<Future<AssertionResult>> verificationFutures = new ArrayList<>();
        
        for (OrderData orderData : orderDataList) {
            Future<AssertionResult> future = executor.submit(() -> {
                // Retrieve from cache
                var retrieved = orderManager.getOrder(SESSION_ID, orderData.orderRef)
                    .orElseThrow(() -> new AssertionError("Order not found: " + orderData.orderRef));
                
                String actualClOrdID = retrieved.clOrdID();
                String expectedClOrdID = orderData.originalClOrdID;
                
                boolean passed = expectedClOrdID.equals(actualClOrdID);
                
                return new AssertionResult(
                    orderData.orderRef,
                    expectedClOrdID,
                    actualClOrdID,
                    passed
                );
            });
            verificationFutures.add(future);
        }
        
        // Collect results
        List<AssertionResult> results = new ArrayList<>();
        for (Future<AssertionResult> future : verificationFutures) {
            results.add(future.get());
        }
        
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS);
        
        // Step 4: ASSERT - Check all results
        List<AssertionResult> failures = results.stream()
            .filter(r -> !r.passed)
            .toList();
        
        if (!failures.isEmpty()) {
            StringBuilder errorMsg = new StringBuilder();
            errorMsg.append("❌ BUG DETECTED: Cached orders were corrupted by external mutations!\n\n");
            errorMsg.append("Failed assertions (").append(failures.size()).append(" out of ").append(results.size()).append("):\n");
            
            for (AssertionResult failure : failures) {
                errorMsg.append(String.format("  Order: %s\n", failure.orderRef));
                errorMsg.append(String.format("    Expected clOrdID: %s\n", failure.expectedClOrdID));
                errorMsg.append(String.format("    Actual clOrdID:   %s (CORRUPTED!)\n", failure.actualClOrdID));
                errorMsg.append("\n");
            }
            
            errorMsg.append("Root Cause: OrderManager storing mutable object references without defensive copying.\n");
            errorMsg.append("Fix: Implement defensive copying in addOrder() method.\n");
            
            fail(errorMsg.toString());
        }
        
        // If we get here, all assertions passed!
        System.out.println("✅ SUCCESS: All " + results.size() + " orders maintained correct clOrdID despite external mutations!");
    }

    @Test
    @DisplayName("Simpler test: Single order mutation should not affect cache")
    void testSingleOrderMutationDoesNotAffectCache() {
        String orderRef = "ORDER_001";
        String originalClOrdID = "ORIGINAL_ID";
        
        // Create and store order
        var order = createTestOrder(originalClOrdID);
        orderManager.addOrder(SESSION_ID, orderRef, order);
        
        // MUTATE the original order (simulating external code)
        order.setClOrdID("MUTATED_ID");
        
        // Retrieve from cache
        var retrieved = orderManager.getOrder(SESSION_ID, orderRef)
            .orElseThrow(() -> new AssertionError("Order not found"));
        
        // ASSERT: Retrieved order should still have original clOrdID
        assertEquals(originalClOrdID, retrieved.clOrdID(),
            "❌ BUG: Cached order was corrupted! Expected '" + originalClOrdID + 
            "' but got '" + retrieved.clOrdID() + "'.\n" +
            "External mutation affected the cached object because OrderManager stored a reference instead of a copy.");
    }

    @Test
    @DisplayName("Multiple sequential mutations should not accumulate in cache")
    void testSequentialMutationsDoNotAccumulate() {
        String orderRef = "ORDER_001";
        String originalClOrdID = "ORIGINAL_ID";
        
        // Create and store order
        var order = createTestOrder(originalClOrdID);
        orderManager.addOrder(SESSION_ID, orderRef, order);
        
        // Perform multiple mutations
        order.setClOrdID("MUTATION_1");
        order.setClOrdID("MUTATION_2");
        order.setClOrdID("MUTATION_3");
        
        // Retrieve from cache
        var retrieved = orderManager.getOrder(SESSION_ID, orderRef)
            .orElseThrow(() -> new AssertionError("Order not found"));
        
        // ASSERT: Should still have original value
        assertEquals(originalClOrdID, retrieved.clOrdID(),
            "❌ BUG: Cached order accumulated mutations! " +
            "This proves OrderManager is storing a reference to a mutable object.");
    }
    
    // Helper method to create test orders
    private DefaultNewOrderSingle createTestOrder(String clOrdID) {
        var order = new DefaultNewOrderSingle();
        order.setClOrdID(clOrdID);
        order.setSenderCompID("SENDER");
        order.setTargetCompID("TARGET");
        order.setSymbol("EUR/USD");
        order.setOrderQty(1000000);
        order.setPrice(1.1001);
        return order;
    }
    
    // Helper classes to hold test data
    private static class OrderData {
        final DefaultNewOrderSingle originalOrder;
        final String orderRef;
        final String originalClOrdID;
        
        OrderData(DefaultNewOrderSingle originalOrder, String orderRef, String originalClOrdID) {
            this.originalOrder = originalOrder;
            this.orderRef = orderRef;
            this.originalClOrdID = originalClOrdID;
        }
    }
    
    private static class AssertionResult {
        final String orderRef;
        final String expectedClOrdID;
        final String actualClOrdID;
        final boolean passed;
        
        AssertionResult(String orderRef, String expectedClOrdID, String actualClOrdID, boolean passed) {
            this.orderRef = orderRef;
            this.expectedClOrdID = expectedClOrdID;
            this.actualClOrdID = actualClOrdID;
            this.passed = passed;
        }
    }
}
```
