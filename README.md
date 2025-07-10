```java

@Test
    void utilizationAndLimit_ShouldBeCorrect_ForBoughtOnly() {
        // Arrange
        CreditCheckRequest req = new CreditCheckRequest();
        Deal deal = new Deal("USD", 40, null, 0, 0); // bought 40 USD
        req.setDeals(List.of(deal));

        CreditParams params = new CreditParams(true, false);
        params.getLimitMap().put("USD", 100.0);

        EarmarkTracker tracker = Mockito.mock(EarmarkTracker.class);
        Mockito.when(tracker.getBoughtAmount(Mockito.any(), Mockito.eq("USD"))).thenReturn(10.0); // earmark 10

        // Act
        boolean result = new YourService().requestIsBelowLimits(req, params, tracker);

        // Assert utilization = 40 + 10 = 50, limit = 100, availability = 50
        assertEquals("50.0", req.getUtilization());
        assertEquals("100.0", req.getLimit());
        assertEquals("50.0", req.getAvailability());
    }

    @Test
    void utilizationAndLimit_ShouldBeCorrect_ForSoldOnly() {
        CreditCheckRequest req = new CreditCheckRequest();
        Deal deal = new Deal(null, 0, "EUR", 70, 0); // sold 70 EUR
        req.setDeals(List.of(deal));

        CreditParams params = new CreditParams(true, false);
        params.getLimitMap().put("EUR", 90.0);

        EarmarkTracker tracker = Mockito.mock(EarmarkTracker.class);
        Mockito.when(tracker.getSoldAmount(Mockito.any(), Mockito.eq("EUR"))).thenReturn(5.0); // earmark 5

        boolean result = new YourService().requestIsBelowLimits(req, params, tracker);

        // utilization = 70 + 5 = 75, limit = 90, availability = 15
        assertEquals("75.0", req.getUtilization());
        assertEquals("90.0", req.getLimit());
        assertEquals("15.0", req.getAvailability());
    }

    @Test
    void utilizationAndLimit_ShouldBeCorrect_WhenBothBoughtAndSold() {
        CreditCheckRequest req = new CreditCheckRequest();
        Deal deal = new Deal("USD", 20, "EUR", 30, 0);
        req.setDeals(List.of(deal));

        CreditParams params = new CreditParams(true, false);
        params.getLimitMap().put("USD", 50.0);
        params.getLimitMap().put("EUR", 70.0);

        EarmarkTracker tracker = Mockito.mock(EarmarkTracker.class);
        Mockito.when(tracker.getBoughtAmount(Mockito.any(), Mockito.eq("USD"))).thenReturn(5.0);
        Mockito.when(tracker.getSoldAmount(Mockito.any(), Mockito.eq("EUR"))).thenReturn(10.0);

        boolean result = new YourService().requestIsBelowLimits(req, params, tracker);

        // utilization = (20+5) + (30+10) = 65, limit = 50 + 70 = 120, availability = 55
        assertEquals("65.0", req.getUtilization());
        assertEquals("120.0", req.getLimit());
        assertEquals("55.0", req.getAvailability());
    }

    @Test
    void utilizationAndLimit_ShouldIncludeCalculatedCurrencyAmount() {
        CreditCheckRequest req = new CreditCheckRequest();
        Deal deal = new Deal(null, 0, null, 0, 25); // only USD calculated
        req.setDeals(List.of(deal));

        CreditParams params = new CreditParams(true, false);
        params.getLimitMap().put("USD", 50.0);

        EarmarkTracker tracker = Mockito.mock(EarmarkTracker.class);
        Mockito.when(tracker.getCalculatedKeyAmount(Mockito.any())).thenReturn(0.0);

        boolean result = new YourService().requestIsBelowLimits(req, params, tracker);

        // utilization = 25, limit = 50, availability = 25
        assertEquals("25.0", req.getUtilization());
        assertEquals("50.0", req.getLimit());
        assertEquals("25.0", req.getAvailability());
    }

    @Test
    void utilizationAndLimit_ShouldHandleZeroDeals() {
        CreditCheckRequest req = new CreditCheckRequest();
        req.setDeals(List.of());

        CreditParams params = new CreditParams(true, false);
        // no limits

        EarmarkTracker tracker = Mockito.mock(EarmarkTracker.class);

        boolean result = new YourService().requestIsBelowLimits(req, params, tracker);

        assertEquals("0.0", req.getUtilization());
        assertEquals("0.0", req.getLimit());
        assertEquals("0.0", req.getAvailability());
    }

```
