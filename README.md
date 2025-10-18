```java

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.List;
import java.util.concurrent.*;
import java.util.stream.IntStream;

@DisplayName("Send multiple orders concurrently under light load")
class CORMockParallelOrderTest extends CORMockClientSimulatorBaseTest {

    @Test
    @DisplayName("Send 10 orders simultaneously")
    void testParallelOrders() throws InterruptedException {
        int orderCount = 10;
        ExecutorService executor = Executors.newFixedThreadPool(orderCount);

        List<Callable<Void>> tasks = IntStream.rangeClosed(1, orderCount)
            .<Callable<Void>>mapToObj(i -> () -> {
                var order = createNewOrderSingle(
                    ExchangeName.PARFX,
                    "EUR",
                    "EURUSD",
                    Side.BUY,
                    SecurityType.FXSPOT,
                    1.1 + (i * 0.0001), // slightly vary price
                    1_000_000L
                );
                order.setTimeInForce(TimeInForce.DAY);

                Log.info("▶ Sending order #{} (price={}, qty={})", i, order.getPrice(), order.getOrderQty());

                clientSimulator.sendTradeMessage(order)
                    .waitForExecutionReport(
                        BASIC_FLOW,
                        Duration.ofSeconds(10),
                        "Order #" + i + " should have pending new, new and trade execution types"
                    );

                return null;
            })
            .toList();

        // 🔥 Run all tasks in parallel and wait for completion
        executor.invokeAll(tasks).forEach(future -> {
            try {
                future.get();
            } catch (Exception e) {
                throw new RuntimeException("Parallel order failed", e);
            }
        });

        executor.shutdown();
        executor.awaitTermination(30, TimeUnit.SECONDS);

        clientSimulator.getPoolStatistics();
    }
}
```
