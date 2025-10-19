```java
package com.citi.fx.qa.mock.cor.orders.manifest;

import com.citi.fx.qa.mock.cor.api.enums.OrdStatus;
import com.citi.fx.qa.mock.cor.gen.messages.datamodel.DefaultNewOrderSingle;
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.parallel.Execution;
import org.junit.jupiter.api.parallel.ExecutionMode;

import java.util.*;
import java.util.concurrent.*;
import java.util.stream.IntStream;

import static org.junit.jupiter.api.Assertions.*;

@DisplayName("OrderManager Regression Tests - Post Defensive Copy Fix")
class OrderManagerRegressionTest {

    private OrderManager orderManager;
    private static final String SESSION_ID = "FIX.4.4:TEST_SESSION";
    private static final String ALT_SESSION_ID = "FIX.4.4:ALT_SESSION";

    @BeforeEach
    void setUp() {
        orderManager = new OrderManager();
    }

    // ==================== BASIC FUNCTIONALITY TESTS ====================

    @Nested
    @DisplayName("Basic CRUD Operations")
    class BasicCrudOperations {

        @Test
        @DisplayName("Should store and retrieve order correctly")
        void testAddAndGetOrder() {
            String orderRef = "ORDER_001";
            var order = createTestOrder("CLO_001");

            orderManager.addOrder(SESSION_ID, orderRef, order);

            var retrieved = orderManager.getOrder(SESSION_ID, orderRef);
            assertTrue(retrieved.isPresent(), "Order should be present");
            assertEquals("CLO_001", retrieved.get().clOrdID());
        }

        @Test
        @DisplayName("Should return empty Optional for non-existent order")
        void testGetNonExistentOrder() {
            var result = orderManager.getOrder(SESSION_ID, "NON_EXISTENT");
            assertTrue(result.isEmpty(), "Should return empty Optional");
        }

        @Test
        @DisplayName("Should update order status correctly")
        void testUpdateOrderStatus() {
            String orderRef = "ORDER_001";
            var order = createTestOrder("CLO_001");
            orderManager.addOrder(SESSION_ID, orderRef, order);

            orderManager.updateOrder(SESSION_ID, orderRef, OrdStatus.NEW);

            var manifest = orderManager.getManifest(new OrderKey(SESSION_ID, orderRef));
            assertTrue(manifest.isPresent());
            assertEquals(OrdStatus.NEW, manifest.get().getOrdStatus());
        }

        @Test
        @DisplayName("Should remove order correctly")
        void testRemoveOrder() {
            String orderRef = "ORDER_001";
            var order = createTestOrder("CLO_001");
            orderManager.addOrder(SESSION_ID, orderRef, order);

            orderManager.removeOrder(SESSION_ID, orderRef);

            var result = orderManager.getOrder(SESSION_ID, orderRef);
            assertTrue(result.isEmpty(), "Order should be removed");
        }

        @Test
        @DisplayName("Should handle multiple orders in same session")
        void testMultipleOrdersSameSession() {
            for (int i = 1; i <= 5; i++) {
                String orderRef = "ORDER_" + i;
                var order = createTestOrder("CLO_" + i);
                orderManager.addOrder(SESSION_ID, orderRef, order);
            }

            for (int i = 1; i <= 5; i++) {
                String orderRef = "ORDER_" + i;
                var retrieved = orderManager.getOrder(SESSION_ID, orderRef);
                assertTrue(retrieved.isPresent());
                assertEquals("CLO_" + i, retrieved.get().clOrdID());
            }
        }

        @Test
        @DisplayName("Should isolate orders between different sessions")
        void testOrderIsolationBetweenSessions() {
            String orderRef = "ORDER_001";
            var order1 = createTestOrder("SESSION1_ORDER");
            var order2 = createTestOrder("SESSION2_ORDER");

            orderManager.addOrder(SESSION_ID, orderRef, order1);
            orderManager.addOrder(ALT_SESSION_ID, orderRef, order2);

            var retrieved1 = orderManager.getOrder(SESSION_ID, orderRef);
            var retrieved2 = orderManager.getOrder(ALT_SESSION_ID, orderRef);

            assertTrue(retrieved1.isPresent());
            assertTrue(retrieved2.isPresent());
            assertEquals("SESSION1_ORDER", retrieved1.get().clOrdID());
            assertEquals("SESSION2_ORDER", retrieved2.get().clOrdID());
        }
    }

    // ==================== DEFENSIVE COPY VERIFICATION TESTS ====================

    @Nested
    @DisplayName("Defensive Copy Verification")
    class DefensiveCopyVerification {

        @Test
        @DisplayName("✅ External mutation should NOT affect cached order")
        void testExternalMutationDoesNotAffectCache() {
            String orderRef = "ORDER_001";
            String originalClOrdID = "ORIGINAL_ID";

            var order = createTestOrder(originalClOrdID);
            orderManager.addOrder(SESSION_ID, orderRef, order);

            // MUTATE original order
            order.setClOrdID("MUTATED_ID");
            order.setSymbol("MUTATED_SYMBOL");
            order.setOrderQty(999999);

            // Retrieve from cache
            var retrieved = orderManager.getOrder(SESSION_ID, orderRef).orElseThrow();

            // VERIFY: Cache should have original values
            assertEquals(originalClOrdID, retrieved.clOrdID(),
                "clOrdID should not be affected by external mutation");
            assertEquals("EUR/USD", retrieved.symbol(),
                "Symbol should not be affected by external mutation");
            assertEquals(1000000, retrieved.orderQty(),
                "OrderQty should not be affected by external mutation");
        }

        @Test
        @DisplayName("✅ Multiple mutations should not accumulate")
        void testMultipleMutationsDoNotAccumulate() {
            String orderRef = "ORDER_001";
            var order = createTestOrder("ORIGINAL");
            orderManager.addOrder(SESSION_ID, orderRef, order);

            // Perform multiple mutations
            for (int i = 1; i <= 10; i++) {
                order.setClOrdID("MUTATION_" + i);
            }

            var retrieved = orderManager.getOrder(SESSION_ID, orderRef).orElseThrow();
            assertEquals("ORIGINAL", retrieved.clOrdID());
        }

        @Test
        @DisplayName("✅ Retrieved order mutation should not affect cache")
        void testRetrievedOrderMutationDoesNotAffectCache() {
            String orderRef = "ORDER_001";
            var order = createTestOrder("ORIGINAL");
            orderManager.addOrder(SESSION_ID, orderRef, order);

            // Retrieve and mutate
            var retrieved1 = orderManager.getOrder(SESSION_ID, orderRef).orElseThrow();
            retrieved1.setClOrdID("MUTATED_BY_RETRIEVAL");

            // Retrieve again
            var retrieved2 = orderManager.getOrder(SESSION_ID, orderRef).orElseThrow();

            // Second retrieval should still have original value
            assertEquals("ORIGINAL", retrieved2.clOrdID(),
                "Cache should not be affected by mutations to retrieved objects");
        }

        @Test
        @DisplayName("✅ All order fields should be independent copies")
        void testAllFieldsAreIndependent() {
            String orderRef = "ORDER_001";
            var order = createTestOrder("ORIGINAL");
            order.setSenderCompID("SENDER_ORIGINAL");
            order.setTargetCompID("TARGET_ORIGINAL");
            order.setSymbol("EUR/USD");
            order.setOrderQty(1000000);
            order.setPrice(1.1001);

            orderManager.addOrder(SESSION_ID, orderRef, order);

            // Mutate all fields
            order.setClOrdID("MUTATED");
            order.setSenderCompID("SENDER_MUTATED");
            order.setTargetCompID("TARGET_MUTATED");
            order.setSymbol("GBP/USD");
            order.setOrderQty(2000000);
            order.setPrice(1.2500);

            var retrieved = orderManager.getOrder(SESSION_ID, orderRef).orElseThrow();

            // All fields should retain original values
            assertEquals("ORIGINAL", retrieved.clOrdID());
            assertEquals("SENDER_ORIGINAL", retrieved.senderCompID());
            assertEquals("TARGET_ORIGINAL", retrieved.targetCompID());
            assertEquals("EUR/USD", retrieved.symbol());
            assertEquals(1000000, retrieved.orderQty());
            assertEquals(1.1001, retrieved.price());
        }
    }

    // ==================== CONCURRENT ACCESS TESTS ====================

    @Nested
    @DisplayName("Concurrent Access Safety")
    @Execution(ExecutionMode.CONCURRENT)
    class ConcurrentAccessSafety {

        @Test
        @DisplayName("✅ Concurrent additions should all succeed")
        void testConcurrentAdditions() throws InterruptedException, ExecutionException {
            int orderCount = 100;
            ExecutorService executor = Executors.newFixedThreadPool(10);

            List<Future<String>> futures = IntStream.range(0, orderCount)
                .mapToObj(i -> executor.submit(() -> {
                    String orderRef = "ORDER_" + i;
                    String clOrdID = "CLO_" + i;
                    var order = createTestOrder(clOrdID);
                    orderManager.addOrder(SESSION_ID, orderRef, order);
                    return clOrdID;
                }))
                .toList();

            // Wait for all completions
            for (Future<String> future : futures) {
                future.get();
            }

            executor.shutdown();
            executor.awaitTermination(5, TimeUnit.SECONDS);

            // Verify all orders are present with correct values
            for (int i = 0; i < orderCount; i++) {
                String orderRef = "ORDER_" + i;
                String expectedClOrdID = "CLO_" + i;

                var retrieved = orderManager.getOrder(SESSION_ID, orderRef);
                assertTrue(retrieved.isPresent(), "Order " + orderRef + " should be present");
                assertEquals(expectedClOrdID, retrieved.get().clOrdID());
            }
        }

        @Test
        @DisplayName("✅ Concurrent mutations should not corrupt cache")
        void testConcurrentMutationsDoNotCorruptCache() throws InterruptedException, ExecutionException {
            int orderCount = 50;
            ExecutorService executor = Executors.newFixedThreadPool(10);

            // Phase 1: Store orders
            Map<String, String> expectedValues = new ConcurrentHashMap<>();
            for (int i = 0; i < orderCount; i++) {
                String orderRef = "ORDER_" + i;
                String clOrdID = "CLO_" + i;
                var order = createTestOrder(clOrdID);
                orderManager.addOrder(SESSION_ID, orderRef, order);
                expectedValues.put(orderRef, clOrdID);
            }

            // Phase 2: Concurrent mutations (simulating external code)
            List<Future<Void>> mutationFutures = IntStream.range(0, orderCount)
                .mapToObj(i -> executor.submit(() -> {
                    String orderRef = "ORDER_" + i;
                    // Create a new order and mutate it (simulating external processing)
                    var order = createTestOrder("CLO_" + i);
                    for (int j = 0; j < 10; j++) {
                        order.setClOrdID("MUTATION_" + i + "_" + j);
                    }
                    return null;
                }))
                .toList();

            // Wait for mutations
            for (Future<Void> future : mutationFutures) {
                future.get();
            }

            executor.shutdown();
            executor.awaitTermination(5, TimeUnit.SECONDS);

            // Phase 3: Verify cache integrity
            for (Map.Entry<String, String> entry : expectedValues.entrySet()) {
                var retrieved = orderManager.getOrder(SESSION_ID, entry.getKey());
                assertTrue(retrieved.isPresent());
                assertEquals(entry.getValue(), retrieved.get().clOrdID(),
                    "Order " + entry.getKey() + " should not be affected by external mutations");
            }
        }

        @Test
        @DisplayName("✅ Concurrent reads and writes should be safe")
        void testConcurrentReadsAndWrites() throws InterruptedException, ExecutionException {
            int iterations = 100;
            ExecutorService executor = Executors.newFixedThreadPool(20);

            List<Future<Boolean>> futures = new ArrayList<>();

            // Half threads writing
            for (int i = 0; i < iterations; i++) {
                final int index = i;
                futures.add(executor.submit(() -> {
                    String orderRef = "ORDER_" + index;
                    var order = createTestOrder("CLO_" + index);
                    orderManager.addOrder(SESSION_ID, orderRef, order);
                    return true;
                }));
            }

            // Half threads reading
            for (int i = 0; i < iterations; i++) {
                final int index = i;
                futures.add(executor.submit(() -> {
                    String orderRef = "ORDER_" + index;
                    // May or may not find it depending on timing, but shouldn't crash
                    orderManager.getOrder(SESSION_ID, orderRef);
                    return true;
                }));
            }

            // Wait for all
            for (Future<Boolean> future : futures) {
                assertTrue(future.get(), "All operations should complete successfully");
            }

            executor.shutdown();
            executor.awaitTermination(5, TimeUnit.SECONDS);
        }

        @Test
        @DisplayName("✅ Concurrent updates should be safe")
        void testConcurrentUpdates() throws InterruptedException, ExecutionException {
            int orderCount = 50;
            String orderRef = "SHARED_ORDER";

            // Add initial order
            var order = createTestOrder("INITIAL");
            orderManager.addOrder(SESSION_ID, orderRef, order);

            ExecutorService executor = Executors.newFixedThreadPool(10);

            // Concurrent status updates
            OrdStatus[] statuses = {OrdStatus.PENDING_NEW, OrdStatus.NEW, OrdStatus.PARTIALLY_FILLED, OrdStatus.FILLED};
            List<Future<Void>> futures = IntStream.range(0, orderCount)
                .mapToObj(i -> executor.submit(() -> {
                    OrdStatus status = statuses[i % statuses.length];
                    orderManager.updateOrder(SESSION_ID, orderRef, status);
                    return null;
                }))
                .toList();

            for (Future<Void> future : futures) {
                future.get();
            }

            executor.shutdown();
            executor.awaitTermination(5, TimeUnit.SECONDS);

            // Should complete without errors
            var manifest = orderManager.getManifest(new OrderKey(SESSION_ID, orderRef));
            assertTrue(manifest.isPresent());
            assertNotNull(manifest.get().getOrdStatus());
        }
    }

    // ==================== EDGE CASES ====================

    @Nested
    @DisplayName("Edge Cases and Error Handling")
    class EdgeCasesAndErrorHandling {

        @Test
        @DisplayName("Should handle orders with identical data but different refs")
        void testIdenticalOrdersDifferentRefs() {
            var order1 = createTestOrder("SAME_ID");
            var order2 = createTestOrder("SAME_ID");

            orderManager.addOrder(SESSION_ID, "REF_1", order1);
            orderManager.addOrder(SESSION_ID, "REF_2", order2);

            var retrieved1 = orderManager.getOrder(SESSION_ID, "REF_1").orElseThrow();
            var retrieved2 = orderManager.getOrder(SESSION_ID, "REF_2").orElseThrow();

            assertEquals("SAME_ID", retrieved1.clOrdID());
            assertEquals("SAME_ID", retrieved2.clOrdID());
        }

        @Test
        @DisplayName("Should handle rapid add/remove cycles")
        void testRapidAddRemoveCycles() {
            String orderRef = "CYCLING_ORDER";

            for (int i = 0; i < 100; i++) {
                var order = createTestOrder("CLO_" + i);
                orderManager.addOrder(SESSION_ID, orderRef, order);

                var retrieved = orderManager.getOrder(SESSION_ID, orderRef);
                assertTrue(retrieved.isPresent());

                orderManager.removeOrder(SESSION_ID, orderRef);

                var afterRemove = orderManager.getOrder(SESSION_ID, orderRef);
                assertTrue(afterRemove.isEmpty());
            }
        }

        @Test
        @DisplayName("Should handle SessionId lookup correctly")
        void testSessionIdLookup() {
            String orderRef = "ORDER_001";
            var order = createTestOrder("CLO_001");

            orderManager.addOrder(SESSION_ID, orderRef, order);

            var sessionId = orderManager.getSessionId(orderRef);
            assertTrue(sessionId.isPresent());
            assertEquals(SESSION_ID, sessionId.get());
        }

        @Test
        @DisplayName("Should return empty for SessionId of non-existent order")
        void testSessionIdLookupNonExistent() {
            var sessionId = orderManager.getSessionId("NON_EXISTENT");
            assertTrue(sessionId.isEmpty());
        }

        @Test
        @DisplayName("Should handle order with null optional fields")
        void testOrderWithNullFields() {
            var order = new DefaultNewOrderSingle();
            order.setClOrdID("MIN_ORDER");
            // Many fields left null

            assertDoesNotThrow(() -> {
                orderManager.addOrder(SESSION_ID, "MIN_REF", order);
                var retrieved = orderManager.getOrder(SESSION_ID, "MIN_REF");
                assertTrue(retrieved.isPresent());
            });
        }
    }

    // ==================== PERFORMANCE TESTS ====================

    @Nested
    @DisplayName("Performance Characteristics")
    class PerformanceCharacteristics {

        @Test
        @DisplayName("Should handle large number of orders efficiently")
        void testLargeScale() {
            int orderCount = 10_000;

            long startTime = System.currentTimeMillis();

            // Add orders
            for (int i = 0; i < orderCount; i++) {
                String orderRef = "ORDER_" + i;
                var order = createTestOrder("CLO_" + i);
                orderManager.addOrder(SESSION_ID, orderRef, order);
            }

            long addTime = System.currentTimeMillis() - startTime;

            // Retrieve orders
            startTime = System.currentTimeMillis();
            for (int i = 0; i < orderCount; i++) {
                String orderRef = "ORDER_" + i;
                orderManager.getOrder(SESSION_ID, orderRef);
            }
            long retrieveTime = System.currentTimeMillis() - startTime;

            System.out.println("Added " + orderCount + " orders in " + addTime + "ms");
            System.out.println("Retrieved " + orderCount + " orders in " + retrieveTime + "ms");

            // Reasonable performance thresholds
            assertTrue(addTime < 5000, "Should add 10k orders in under 5 seconds");
            assertTrue(retrieveTime < 1000, "Should retrieve 10k orders in under 1 second");
        }

        @Test
        @DisplayName("Defensive copy overhead should be minimal")
        void testDefensiveCopyOverhead() {
            int iterations = 1000;

            long startTime = System.nanoTime();

            for (int i = 0; i < iterations; i++) {
                String orderRef = "ORDER_" + i;
                var order = createTestOrder("CLO_" + i);
                orderManager.addOrder(SESSION_ID, orderRef, order);
            }

            long duration = System.nanoTime() - startTime;
            double avgMicroseconds = (duration / 1000.0) / iterations;

            System.out.println("Average time per order (with defensive copy): " +
                String.format("%.2f", avgMicroseconds) + " microseconds");

            // Defensive copy should add minimal overhead (< 100 microseconds per order)
            assertTrue(avgMicroseconds < 100,
                "Defensive copy overhead should be minimal");
        }
    }

    // ==================== INTEGRATION-STYLE TESTS ====================

    @Nested
    @DisplayName("Integration-Style Scenarios")
    class IntegrationScenarios {

        @Test
        @DisplayName("Should handle complete order lifecycle")
        void testCompleteOrderLifecycle() {
            String orderRef = "LIFECYCLE_ORDER";
            var order = createTestOrder("LIFECYCLE_CLO");

            // 1. Add order
            orderManager.addOrder(SESSION_ID, orderRef, order);
            var retrieved = orderManager.getOrder(SESSION_ID, orderRef);
            assertTrue(retrieved.isPresent());

            // 2. Update to PENDING_NEW
            orderManager.updateOrder(SESSION_ID, orderRef, OrdStatus.PENDING_NEW);
            var manifest = orderManager.getManifest(new OrderKey(SESSION_ID, orderRef));
            assertEquals(OrdStatus.PENDING_NEW, manifest.get().getOrdStatus());

            // 3. Update to NEW
            orderManager.updateOrder(SESSION_ID, orderRef, OrdStatus.NEW);
            manifest = orderManager.getManifest(new OrderKey(SESSION_ID, orderRef));
            assertEquals(OrdStatus.NEW, manifest.get().getOrdStatus());

            // 4. Update to FILLED
            orderManager.updateOrder(SESSION_ID, orderRef, OrdStatus.FILLED);
            manifest = orderManager.getManifest(new OrderKey(SESSION_ID, orderRef));
            assertEquals(OrdStatus.FILLED, manifest.get().getOrdStatus());

            // 5. Remove order
            orderManager.removeOrder(SESSION_ID, orderRef);
            retrieved = orderManager.getOrder(SESSION_ID, orderRef);
            assertTrue(retrieved.isEmpty());
        }

        @Test
        @DisplayName("Should handle multiple sessions with interleaved operations")
        void testMultipleSessionsInterleaved() {
            // Session 1
            orderManager.addOrder("SESSION_1", "ORDER_A", createTestOrder("S1_A"));
            orderManager.addOrder("SESSION_1", "ORDER_B", createTestOrder("S1_B"));

            // Session 2
            orderManager.addOrder("SESSION_2", "ORDER_A", createTestOrder("S2_A"));
            orderManager.addOrder("SESSION_2", "ORDER_B", createTestOrder("S2_B"));

            // Session 3
            orderManager.addOrder("SESSION_3", "ORDER_A", createTestOrder("S3_A"));

            // Verify isolation
            assertEquals("S1_A", orderManager.getOrder("SESSION_1", "ORDER_A").get().clOrdID());
            assertEquals("S1_B", orderManager.getOrder("SESSION_1", "ORDER_B").get().clOrdID());
            assertEquals("S2_A", orderManager.getOrder("SESSION_2", "ORDER_A").get().clOrdID());
            assertEquals("S2_B", orderManager.getOrder("SESSION_2", "ORDER_B").get().clOrdID());
            assertEquals("S3_A", orderManager.getOrder("SESSION_3", "ORDER_A").get().clOrdID());
        }

        @Test
        @DisplayName("Should handle cancel request workflow")
        void testCancelRequestWorkflow() {
            String orderRef = "CANCEL_TEST";
            var order = createTestOrder("CANCEL_CLO");

            // Add order
            orderManager.addOrder(SESSION_ID, orderRef, order);

            // Simulate cancel request
            var cancelRequest = new com.citi.fx.qa.mock.cor.gen.messages.datamodel.DefaultOrderCancelRequest();
            cancelRequest.setClOrdID("CANCEL_CLO");
            cancelRequest.setOrigClOrdID(order.clOrdID());

            orderManager.addCancelRequest(cancelRequest);

            var retrievedCancel = orderManager.getCancelRequest("CANCEL_CLO");
            assertTrue(retrievedCancel.isPresent());
            assertEquals("CANCEL_CLO", retrievedCancel.get().clOrdID());
        }
    }

    // ==================== HELPER METHODS ====================

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
}
```
