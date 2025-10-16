```java
public static Stream<LiquidityPoolTestData> liquidityPoolTestDataStream() {
    return Stream.of(
        new LiquidityPoolTestData(EBS,     TargetStrategy.FUS_TWAP,         "USDJPY", "USD", BUY),
        new LiquidityPoolTestData(TRM,     TargetStrategy.FUS_VOL_TRACKER,  "EURUSD", "EUR", SELL),
        new LiquidityPoolTestData(CNX,     TargetStrategy.FUS_PEG,          "AUDUSD", "USD", BUY),
        new LiquidityPoolTestData(CME,     TargetStrategy.FUS_VWAP,         "GBPUSD", "GBP", SELL),
        new LiquidityPoolTestData(PARFX,   TargetStrategy.FUS_ARRIVAL,      "USDCNH", "CNY", BUY),
        new LiquidityPoolTestData(LMAF,    TargetStrategy.FUS_TWAP,         "USDKRW", "KRW", SELL),
        new LiquidityPoolTestData(HENNING, TargetStrategy.FUS_VOL_TRACKER,  "USDTWD", "TWD", BUY),
        new LiquidityPoolTestData(VIRTU,   TargetStrategy.FUS_PEG,          "USDPEN", "PEN", SELL),
        new LiquidityPoolTestData(FXALL,   TargetStrategy.FUS_VWAP,         "EURUSD", "EUR", BUY),
        new LiquidityPoolTestData(FM,      TargetStrategy.FUS_ARRIVAL,      "GBPUSD", "GBP", SELL)
    );
}
```
