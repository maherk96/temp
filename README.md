```java
package com.citi.fx.qa.mock.cor.orders.manifest;

import com.citi.fx.qa.mock.cor.api.enums.OrdStatus;
import com.citi.fx.qa.mock.cor.gen.messages.datamodel.DefaultNewOrderSingle;
import org.junit.jupiter.api.*;

import java.util.*;
import java.util.concurrent.*;
import java.util.stream.IntStream;

import static org.junit.jupiter.api.Assertions.*;

@DisplayName("OrderManager - Comprehensive Test Suite")
class OrderManagerTest {

    private OrderManager orderManager;
    private static final String PRIMARY_SESSION = "FIX.4.4:QA1_ALGO-UAT-TRADE";
    private static final String SECONDARY_SESSION = "FIX.4.4:QA2_ALGO-UAT-TRADE";

    @BeforeEach
    void setUp() {
        orderManager = new OrderManager();
    }

    // ==================== BASIC OPERATIONS ====================

    @Nested
    @DisplayName("Order Storage and Retrieval")
    class OrderStorageAndRetrieval {

        @Test
        @DisplayName("Should successfully store and retrieve a single order")
        void shouldStoreAndRetrieveSingleOrder() {
            var order = createMarketOrder("O7SCDDT00001H1P", "EUR/USD", 1000000, 1.1001);
            
            orderManager.addOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", order);
            
            var retrieved = orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT00001H1P");
            
            assertTrue(retrieved.isPresent());
            assertEquals("O7SCDDT00001H1P", retrieved.get().clOrdID());
            assertEquals("EUR/USD", retrieved.get().symbol());
            assertEquals(1000000, retrieved.get().orderQty());
        }

        @Test
        @DisplayName("Should return empty when order does not exist")
        void shouldReturnEmptyForNonExistentOrder() {
            var result = orderManager.getOrder(PRIMARY_SESSION, "NONEXISTENT123");
            
            assertTrue(result.isEmpty());
        }

        @Test
        @DisplayName("Should store multiple orders independently")
        void shouldStoreMultipleOrdersIndependently() {
            var order1 = createMarketOrder("O7SCDDT00001H1P", "EUR/USD", 1000000, 1.1001);
            var order2 = createMarketOrder("O7SCDDT00002H1P", "GBP/USD", 2000000, 1.2500);
            var order3 = createMarketOrder("O7SCDDT00003H1P", "USD/JPY", 1500000, 110.50);
            
            orderManager.addOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", order1);
            orderManager.addOrder(PRIMARY_SESSION, "O7SCDDT00002H1P", order2);
            orderManager.addOrder(PRIMARY_SESSION, "O7SCDDT00003H1P", order3);
            
            var retrieved1 = orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT00001H1P").orElseThrow();
            var retrieved2 = orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT00002H1P").orElseThrow();
            var retrieved3 = orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT00003H1P").orElseThrow();
            
            assertEquals("EUR/USD", retrieved1.symbol());
            assertEquals("GBP/USD", retrieved2.symbol());
            assertEquals("USD/JPY", retrieved3.symbol());
            
            assertEquals(1000000, retrieved1.orderQty());
            assertEquals(2000000, retrieved2.orderQty());
            assertEquals(1500000, retrieved3.orderQty());
        }

        @Test
        @DisplayName("Should handle orders across different sessions")
        void shouldHandleOrdersAcrossDifferentSessions() {
            var order1 = createMarketOrder("O7SCDDT00001H1P", "EUR/USD", 1000000, 1.1001);
            var order2 = createMarketOrder("O7SCDDT00002H1P", "EUR/USD", 1000000, 1.1001);
            
            orderManager.addOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", order1);
            orderManager.addOrder(SECONDARY_SESSION, "O7SCDDT00002H1P", order2);
            
            var fromPrimary = orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT00001H1P");
            var fromSecondary = orderManager.getOrder(SECONDARY_SESSION, "O7SCDDT00002H1P");
            var crossCheck1 = orderManager.getOrder(SECONDARY_SESSION, "O7SCDDT00001H1P");
            var crossCheck2 = orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT00002H1P");
            
            assertTrue(fromPrimary.isPresent());
            assertTrue(fromSecondary.isPresent());
            assertTrue(crossCheck1.isEmpty());
            assertTrue(crossCheck2.isEmpty());
        }

        @Test
        @DisplayName("Should retrieve correct session ID for order reference")
        void shouldRetrieveCorrectSessionId() {
            var order = createMarketOrder("O7SCDDT00001H1P", "EUR/USD", 1000000, 1.1001);
            
            orderManager.addOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", order);
            
            var sessionId = orderManager.getSessionId("O7SCDDT00001H1P");
            
            assertTrue(sessionId.isPresent());
            assertEquals(PRIMARY_SESSION, sessionId.get());
        }
    }

    // ==================== ORDER LIFECYCLE ====================

    @Nested
    @DisplayName("Order Lifecycle Management")
    class OrderLifecycleManagement {

        @Test
        @DisplayName("Should track order status updates through lifecycle")
        void shouldTrackOrderStatusThroughLifecycle() {
            var order = createMarketOrder("O7SCDDT00001H1P", "EUR/USD", 1000000, 1.1001);
            
            orderManager.addOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", order);
            
            orderManager.updateOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", OrdStatus.PENDING_NEW);
            var manifest1 = orderManager.getManifest(new OrderKey(PRIMARY_SESSION, "O7SCDDT00001H1P")).orElseThrow();
            assertEquals(OrdStatus.PENDING_NEW, manifest1.getOrdStatus());
            
            orderManager.updateOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", OrdStatus.NEW);
            var manifest2 = orderManager.getManifest(new OrderKey(PRIMARY_SESSION, "O7SCDDT00001H1P")).orElseThrow();
            assertEquals(OrdStatus.NEW, manifest2.getOrdStatus());
            
            orderManager.updateOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", OrdStatus.PARTIALLY_FILLED);
            var manifest3 = orderManager.getManifest(new OrderKey(PRIMARY_SESSION, "O7SCDDT00001H1P")).orElseThrow();
            assertEquals(OrdStatus.PARTIALLY_FILLED, manifest3.getOrdStatus());
            
            orderManager.updateOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", OrdStatus.FILLED);
            var manifest4 = orderManager.getManifest(new OrderKey(PRIMARY_SESSION, "O7SCDDT00001H1P")).orElseThrow();
            assertEquals(OrdStatus.FILLED, manifest4.getOrdStatus());
        }

        @Test
        @DisplayName("Should successfully remove order")
        void shouldSuccessfullyRemoveOrder() {
            var order = createMarketOrder("O7SCDDT00001H1P", "EUR/USD", 1000000, 1.1001);
            
            orderManager.addOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", order);
            assertTrue(orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT00001H1P").isPresent());
            
            orderManager.removeOrder(PRIMARY_SESSION, "O7SCDDT00001H1P");
            
            assertTrue(orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT00001H1P").isEmpty());
        }

        @Test
        @DisplayName("Should handle complete order workflow from creation to removal")
        void shouldHandleCompleteOrderWorkflow() {
            var order = createMarketOrder("O7SCDDT00001H1P", "EUR/USD", 1000000, 1.1001);
            
            // Submit
            orderManager.addOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", order);
            var submitted = orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT00001H1P");
            assertTrue(submitted.isPresent());
            
            // Acknowledge
            orderManager.updateOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", OrdStatus.NEW);
            
            // Fill
            orderManager.updateOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", OrdStatus.FILLED);
            var manifest = orderManager.getManifest(new OrderKey(PRIMARY_SESSION, "O7SCDDT00001H1P")).orElseThrow();
            assertEquals(OrdStatus.FILLED, manifest.getOrdStatus());
            
            // Clean up
            orderManager.removeOrder(PRIMARY_SESSION, "O7SCDDT00001H1P");
            assertTrue(orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT00001H1P").isEmpty());
        }
    }

    // ==================== DATA INTEGRITY ====================

    @Nested
    @DisplayName("Data Integrity and Consistency")
    class DataIntegrityAndConsistency {

        @Test
        @DisplayName("Should maintain order data integrity during lifecycle")
        void shouldMaintainOrderDataIntegrityDuringLifecycle() {
            var order = createMarketOrder("O7SCDDT00001H1P", "EUR/USD", 1000000, 1.1001);
            
            orderManager.addOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", order);
            
            // Verify initial state
            var initial = orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT00001H1P").orElseThrow();
            assertEquals("O7SCDDT00001H1P", initial.clOrdID());
            assertEquals("EUR/USD", initial.symbol());
            assertEquals(1000000, initial.orderQty());
            
            // Update status multiple times
            orderManager.updateOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", OrdStatus.PENDING_NEW);
            orderManager.updateOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", OrdStatus.NEW);
            orderManager.updateOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", OrdStatus.FILLED);
            
            // Verify order data unchanged
            var afterUpdates = orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT00001H1P").orElseThrow();
            assertEquals("O7SCDDT00001H1P", afterUpdates.clOrdID());
            assertEquals("EUR/USD", afterUpdates.symbol());
            assertEquals(1000000, afterUpdates.orderQty());
            assertEquals(1.1001, afterUpdates.price());
        }

        @Test
        @DisplayName("Should maintain data consistency across multiple retrievals")
        void shouldMaintainDataConsistencyAcrossRetrievals() {
            var order = createMarketOrder("O7SCDDT00001H1P", "EUR/USD", 1000000, 1.1001);
            
            orderManager.addOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", order);
            
            // Multiple retrievals
            var retrieval1 = orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT00001H1P").orElseThrow();
            var retrieval2 = orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT00001H1P").orElseThrow();
            var retrieval3 = orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT00001H1P").orElseThrow();
            
            // All should have same data
            assertEquals("O7SCDDT00001H1P", retrieval1.clOrdID());
            assertEquals("O7SCDDT00001H1P", retrieval2.clOrdID());
            assertEquals("O7SCDDT00001H1P", retrieval3.clOrdID());
            
            assertEquals("EUR/USD", retrieval1.symbol());
            assertEquals("EUR/USD", retrieval2.symbol());
            assertEquals("EUR/USD", retrieval3.symbol());
            
            assertEquals(1000000, retrieval1.orderQty());
            assertEquals(1000000, retrieval2.orderQty());
            assertEquals(1000000, retrieval3.orderQty());
        }

        @Test
        @DisplayName("Should preserve all order fields accurately")
        void shouldPreserveAllOrderFieldsAccurately() {
            var order = new DefaultNewOrderSingle();
            order.setClOrdID("O7SCDDT00001H1P");
            order.setClOrdLinkID("LINK123");
            order.setSenderCompID("QA1_ALGO-UAT-TRADE");
            order.setTargetCompID("COR-MOCK-NAM-1");
            order.setSymbol("EUR/USD");
            order.setOrderQty(1000000);
            order.setPrice(1.1001);
            order.setCurrency("EUR");
            
            orderManager.addOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", order);
            
            var retrieved = orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT00001H1P").orElseThrow();
            
            assertEquals("O7SCDDT00001H1P", retrieved.clOrdID());
            assertEquals("LINK123", retrieved.clOrdLinkID());
            assertEquals("QA1_ALGO-UAT-TRADE", retrieved.senderCompID());
            assertEquals("COR-MOCK-NAM-1", retrieved.targetCompID());
            assertEquals("EUR/USD", retrieved.symbol());
            assertEquals(1000000, retrieved.orderQty());
            assertEquals(1.1001, retrieved.price());
            assertEquals("EUR", retrieved.currency());
        }

        @Test
        @DisplayName("Should handle orders with same reference in different sessions")
        void shouldHandleOrdersWithSameReferenceInDifferentSessions() {
            var order1 = createMarketOrder("O7SCDDT00001H1P", "EUR/USD", 1000000, 1.1001);
            var order2 = createMarketOrder("O7SCDDT00001H1P", "GBP/USD", 2000000, 1.2500);
            
            orderManager.addOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", order1);
            orderManager.addOrder(SECONDARY_SESSION, "O7SCDDT00001H1P", order2);
            
            var fromPrimary = orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT00001H1P").orElseThrow();
            var fromSecondary = orderManager.getOrder(SECONDARY_SESSION, "O7SCDDT00001H1P").orElseThrow();
            
            // Should retrieve different orders
            assertEquals("EUR/USD", fromPrimary.symbol());
            assertEquals("GBP/USD", fromSecondary.symbol());
            assertEquals(1000000, fromPrimary.orderQty());
            assertEquals(2000000, fromSecondary.orderQty());
        }
    }

    // ==================== CONCURRENT OPERATIONS ====================

    @Nested
    @DisplayName("Concurrent Order Processing")
    class ConcurrentOrderProcessing {

        @Test
        @DisplayName("Should handle concurrent order submissions")
        void shouldHandleConcurrentOrderSubmissions() throws InterruptedException, ExecutionException {
            int orderCount = 100;
            ExecutorService executor = Executors.newFixedThreadPool(10);
            
            List<Future<Void>> futures = IntStream.range(1, orderCount + 1)
                .mapToObj(i -> executor.submit(() -> {
                    String orderRef = String.format("O7SCDDT%05dH1P", i);
                    var order = createMarketOrder(orderRef, "EUR/USD", 1000000 + i, 1.1001 + (i * 0.0001));
                    orderManager.addOrder(PRIMARY_SESSION, orderRef, order);
                    return null;
                }))
                .toList();
            
            for (Future<Void> future : futures) {
                future.get();
            }
            
            executor.shutdown();
            executor.awaitTermination(5, TimeUnit.SECONDS);
            
            // Verify all orders stored correctly
            for (int i = 1; i <= orderCount; i++) {
                String orderRef = String.format("O7SCDDT%05dH1P", i);
                var retrieved = orderManager.getOrder(PRIMARY_SESSION, orderRef);
                assertTrue(retrieved.isPresent(), "Order " + orderRef + " should exist");
                assertEquals(orderRef, retrieved.get().clOrdID());
                assertEquals(1000000 + i, retrieved.get().orderQty());
            }
        }

        @Test
        @DisplayName("Should handle concurrent order lookups")
        void shouldHandleConcurrentOrderLookups() throws InterruptedException, ExecutionException {
            // Setup: Create orders
            for (int i = 1; i <= 50; i++) {
                String orderRef = String.format("O7SCDDT%05dH1P", i);
                var order = createMarketOrder(orderRef, "EUR/USD", 1000000, 1.1001);
                orderManager.addOrder(PRIMARY_SESSION, orderRef, order);
            }
            
            ExecutorService executor = Executors.newFixedThreadPool(20);
            
            // Concurrent reads
            List<Future<Boolean>> futures = IntStream.range(1, 51)
                .mapToObj(i -> executor.submit(() -> {
                    String orderRef = String.format("O7SCDDT%05dH1P", i);
                    var order = orderManager.getOrder(PRIMARY_SESSION, orderRef);
                    return order.isPresent() && orderRef.equals(order.get().clOrdID());
                }))
                .toList();
            
            for (Future<Boolean> future : futures) {
                assertTrue(future.get(), "All lookups should succeed");
            }
            
            executor.shutdown();
            executor.awaitTermination(5, TimeUnit.SECONDS);
        }

        @Test
        @DisplayName("Should handle concurrent status updates")
        void shouldHandleConcurrentStatusUpdates() throws InterruptedException, ExecutionException {
            int orderCount = 50;
            
            // Setup orders
            for (int i = 1; i <= orderCount; i++) {
                String orderRef = String.format("O7SCDDT%05dH1P", i);
                var order = createMarketOrder(orderRef, "EUR/USD", 1000000, 1.1001);
                orderManager.addOrder(PRIMARY_SESSION, orderRef, order);
            }
            
            ExecutorService executor = Executors.newFixedThreadPool(10);
            
            // Concurrent updates
            OrdStatus[] statuses = {OrdStatus.PENDING_NEW, OrdStatus.NEW, OrdStatus.PARTIALLY_FILLED, OrdStatus.FILLED};
            List<Future<Void>> futures = IntStream.range(1, orderCount + 1)
                .mapToObj(i -> executor.submit(() -> {
                    String orderRef = String.format("O7SCDDT%05dH1P", i);
                    OrdStatus status = statuses[i % statuses.length];
                    orderManager.updateOrder(PRIMARY_SESSION, orderRef, status);
                    return null;
                }))
                .toList();
            
            for (Future<Void> future : futures) {
                future.get();
            }
            
            executor.shutdown();
            executor.awaitTermination(5, TimeUnit.SECONDS);
            
            // Verify all have a status
            for (int i = 1; i <= orderCount; i++) {
                String orderRef = String.format("O7SCDDT%05dH1P", i);
                var manifest = orderManager.getManifest(new OrderKey(PRIMARY_SESSION, orderRef));
                assertTrue(manifest.isPresent());
                assertNotNull(manifest.get().getOrdStatus());
            }
        }

        @Test
        @DisplayName("Should handle mixed concurrent operations")
        void shouldHandleMixedConcurrentOperations() throws InterruptedException, ExecutionException {
            ExecutorService executor = Executors.newFixedThreadPool(15);
            List<Future<Void>> futures = new ArrayList<>();
            
            // Mix of adds, gets, and updates
            for (int i = 1; i <= 100; i++) {
                final int index = i;
                
                if (i % 3 == 0) {
                    // Add
                    futures.add(executor.submit(() -> {
                        String orderRef = String.format("O7SCDDT%05dH1P", index);
                        var order = createMarketOrder(orderRef, "EUR/USD", 1000000, 1.1001);
                        orderManager.addOrder(PRIMARY_SESSION, orderRef, order);
                        return null;
                    }));
                } else if (i % 3 == 1) {
                    // Get
                    futures.add(executor.submit(() -> {
                        String orderRef = String.format("O7SCDDT%05dH1P", index);
                        orderManager.getOrder(PRIMARY_SESSION, orderRef);
                        return null;
                    }));
                } else {
                    // Update
                    futures.add(executor.submit(() -> {
                        String orderRef = String.format("O7SCDDT%05dH1P", index);
                        orderManager.updateOrder(PRIMARY_SESSION, orderRef, OrdStatus.NEW);
                        return null;
                    }));
                }
            }
            
            for (Future<Void> future : futures) {
                assertDoesNotThrow(() -> future.get());
            }
            
            executor.shutdown();
            executor.awaitTermination(5, TimeUnit.SECONDS);
        }
    }

    // ==================== CANCEL REQUEST MANAGEMENT ====================

    @Nested
    @DisplayName("Cancel Request Management")
    class CancelRequestManagement {

        @Test
        @DisplayName("Should store and retrieve cancel request")
        void shouldStoreAndRetrieveCancelRequest() {
            var cancelRequest = new com.citi.fx.qa.mock.cor.gen.messages.datamodel.DefaultOrderCancelRequest();
            cancelRequest.setClOrdID("CANCEL_O7SCDDT00001H1P");
            cancelRequest.setOrigClOrdID("O7SCDDT00001H1P");
            
            orderManager.addCancelRequest(cancelRequest);
            
            var retrieved = orderManager.getCancelRequest("CANCEL_O7SCDDT00001H1P");
            
            assertTrue(retrieved.isPresent());
            assertEquals("CANCEL_O7SCDDT00001H1P", retrieved.get().clOrdID());
            assertEquals("O7SCDDT00001H1P", retrieved.get().origClOrdID());
        }

        @Test
        @DisplayName("Should handle cancel workflow")
        void shouldHandleCancelWorkflow() {
            // Original order
            var order = createMarketOrder("O7SCDDT00001H1P", "EUR/USD", 1000000, 1.1001);
            orderManager.addOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", order);
            orderManager.updateOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", OrdStatus.NEW);
            
            // Cancel request
            var cancelRequest = new com.citi.fx.qa.mock.cor.gen.messages.datamodel.DefaultOrderCancelRequest();
            cancelRequest.setClOrdID("CANCEL_O7SCDDT00001H1P");
            cancelRequest.setOrigClOrdID("O7SCDDT00001H1P");
            orderManager.addCancelRequest(cancelRequest);
            
            // Update to cancelled
            orderManager.updateOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", OrdStatus.CANCELED);
            
            var manifest = orderManager.getManifest(new OrderKey(PRIMARY_SESSION, "O7SCDDT00001H1P")).orElseThrow();
            assertEquals(OrdStatus.CANCELED, manifest.getOrdStatus());
        }
    }

    // ==================== EDGE CASES ====================

    @Nested
    @DisplayName("Edge Cases and Boundary Conditions")
    class EdgeCasesAndBoundaryConditions {

        @Test
        @DisplayName("Should handle rapid successive operations on same order")
        void shouldHandleRapidSuccessiveOperations() {
            String orderRef = "O7SCDDT00001H1P";
            
            for (int i = 0; i < 100; i++) {
                var order = createMarketOrder(orderRef, "EUR/USD", 1000000 + i, 1.1001);
                orderManager.addOrder(PRIMARY_SESSION, orderRef, order);
                
                var retrieved = orderManager.getOrder(PRIMARY_SESSION, orderRef);
                assertTrue(retrieved.isPresent());
                
                orderManager.removeOrder(PRIMARY_SESSION, orderRef);
                
                var afterRemove = orderManager.getOrder(PRIMARY_SESSION, orderRef);
                assertTrue(afterRemove.isEmpty());
            }
        }

        @Test
        @DisplayName("Should handle orders with minimal required fields")
        void shouldHandleOrdersWithMinimalFields() {
            var minimalOrder = new DefaultNewOrderSingle();
            minimalOrder.setClOrdID("MINIMAL_ORDER");
            
            assertDoesNotThrow(() -> {
                orderManager.addOrder(PRIMARY_SESSION, "MINIMAL_ORDER", minimalOrder);
                var retrieved = orderManager.getOrder(PRIMARY_SESSION, "MINIMAL_ORDER");
                assertTrue(retrieved.isPresent());
            });
        }

        @Test
        @DisplayName("Should handle large order quantities and prices")
        void shouldHandleLargeOrderQuantitiesAndPrices() {
            var largeOrder = createMarketOrder(
                "O7SCDDT00001H1P", 
                "EUR/USD", 
                Integer.MAX_VALUE, 
                999999.9999
            );
            
            orderManager.addOrder(PRIMARY_SESSION, "O7SCDDT00001H1P", largeOrder);
            
            var retrieved = orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT00001H1P").orElseThrow();
            
            assertEquals(Integer.MAX_VALUE, retrieved.orderQty());
            assertEquals(999999.9999, retrieved.price(), 0.0001);
        }

        @Test
        @DisplayName("Should handle special characters in order fields")
        void shouldHandleSpecialCharactersInOrderFields() {
            var order = new DefaultNewOrderSingle();
            order.setClOrdID("O7SCDDT-00001-H1P");
            order.setSymbol("EUR/USD.SPOT");
            order.setSenderCompID("SENDER@COMP#123");
            
            orderManager.addOrder(PRIMARY_SESSION, "O7SCDDT-00001-H1P", order);
            
            var retrieved = orderManager.getOrder(PRIMARY_SESSION, "O7SCDDT-00001-H1P").orElseThrow();
            
            assertEquals("O7SCDDT-00001-H1P", retrieved.clOrdID());
            assertEquals("EUR/USD.SPOT", retrieved.symbol());
            assertEquals("SENDER@COMP#123", retrieved.senderCompID());
        }
    }

    // ==================== PERFORMANCE ====================

    @Nested
    @DisplayName("Performance Characteristics")
    class PerformanceCharacteristics {

        @Test
        @DisplayName("Should handle high volume of orders efficiently")
        void shouldHandleHighVolumeEfficiently() {
            int orderCount = 10_000;
            
            long startAdd = System.currentTimeMillis();
            for (int i = 1; i <= orderCount; i++) {
                String orderRef = String.format("O7SCDDT%05dH1P", i);
                var order = createMarketOrder(orderRef, "EUR/USD", 1000000, 1.1001);
                orderManager.addOrder(PRIMARY_SESSION, orderRef, order);
            }
            long addTime = System.currentTimeMillis() - startAdd;
            
            long startRetrieve = System.currentTimeMillis();
            for (int i = 1; i <= orderCount; i++) {
                String orderRef = String.format("O7SCDDT%05dH1P", i);
                orderManager.getOrder(PRIMARY_SESSION, orderRef);
            }
            long retrieveTime = System.currentTimeMillis() - startRetrieve;
            
            System.out.println("Performance: Added " + orderCount + " orders in " + addTime + "ms");
            System.out.println("Performance: Retrieved " + orderCount + " orders in " + retrieveTime + "ms");
            
            assertTrue(addTime < 5000, "Should add 10k orders in under 5 seconds");
            assertTrue(retrieveTime < 1000, "Should retrieve 10k orders in under 1 second");
        }

        @Test
        @DisplayName("Should maintain consistent retrieval performance")
        void shouldMaintainConsistentRetrievalPerformance() {
            // Add 1000 orders
            for (int i = 1; i <= 1000; i++) {
                String orderRef = String.format("O7SCDDT%05dH1P", i);
                var order = createMarketOrder(orderRef, "EUR/USD", 1000000, 1.1001);
                orderManager.addOrder(PRIMARY_SESSION, orderRef, order);
            }
            
            // Measure retrieval times
            List<Long> retrievalTimes = new ArrayList<>();
            for (int i = 1; i <= 1000; i++) {
                String orderRef = String.format("O7SCDDT%05dH1P", i);
                long start = System.nanoTime();
                orderManager.getOrder(PRIMARY_SESSION, orderRef);
                long duration = System.nanoTime() - start;
                retrievalTimes.add(duration);
            }
            
            double avgMicros = retrievalTimes.stream()
                .mapToLong(Long::longValue)
                .average()
                .orElse(0.0) / 1000.0;
            
            System.out.println("Average retrieval time: " + String.format("%.2f", avgMicros) + " microseconds");
            
            assertTrue(avgMicros < 100, "Average retrieval should be under 100 microseconds");
        }
    }

    // ==================== HELPER METHODS ====================

    private DefaultNewOrderSingle createMarketOrder(String clOrdID, String symbol, int quantity, double price) {
        var order = new DefaultNewOrderSingle();
        order.setClOrdID(clOrdID);
        order.setSenderCompID("QA1_ALGO-UAT-TRADE");
        order.setTargetCompID("COR-MOCK-NAM-1");
        order.setSymbol(symbol);
        order.setOrderQty(quantity);
        order.setPrice(price);
        return order;
    }
}

```
