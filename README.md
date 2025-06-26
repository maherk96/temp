# temp

```java
task runTradeServiceTests(type: Test) {
    useJUnitPlatform()

    systemProperty "launchID", "AlgoFusionBookingRegression"
    systemProperty "qap.regression", true

    include '**/TradeServiceTest.class'
    include '**/TradeServiceTest_1.class'
    include '**/TradeServiceTest_2.class'

    maxParallelForks = 1
}
```
