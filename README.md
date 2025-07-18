```java
import org.agrona.MutableDirectBuffer;
import org.agrona.DirectBuffer;
import org.agrona.concurrent.UnsafeBuffer;
import java.nio.ByteBuffer;
import java.nio.charset.StandardCharsets;


public class SbeMassUIQuoteExample {

    public static void main(String[] args) {
        // Create a buffer
        ByteBuffer byteBuffer = ByteBuffer.allocateDirect(4096);
        MutableDirectBuffer buffer = new UnsafeBuffer(byteBuffer);

        // Encode the message
        encode(buffer);

        // Decode the message
        decode(buffer);
    }

    private static void encode(MutableDirectBuffer buffer) {
        MessageHeaderEncoder headerEncoder = new MessageHeaderEncoder();
        MassUIQuoteEncoder quoteEncoder = new MassUIQuoteEncoder();

        // Wrap and apply header
        int offset = 0;
        quoteEncoder.wrapAndApplyHeader(buffer, offset, headerEncoder);

        quoteEncoder.quoteId(101);
        quoteEncoder.sendingTime(System.currentTimeMillis());
        quoteEncoder.senderCompID(501);
        quoteEncoder.senderSubID(502);
        quoteEncoder.senderLocationID(503);

        // Currency Pair Pricing group (1 entry)
        MassUIQuoteEncoder.CurrencyPairPricingEncoder pricingGroup = quoteEncoder.currencyPairPricingCount(1);
        pricingGroup.next()
                .currencyPair("EURUSD")
                .pip(0.0001)
                .settleDate(System.currentTimeMillis())
                .maturityDate(System.currentTimeMillis() + 86400000)
                .spotForwardBidPoints(0.00002)
                .spotForwardAskPoints(0.00003)
                .quoteType(QuoteType.TRADEABLE);

        // Driving Currency Pair
        MassUIQuoteEncoder.DrivingCurrencyPairsEncoder drivingGroup = quoteEncoder.drivingCurrencyPairsCount(1);
        drivingGroup.next().currencyPair("USD");

        // Adjustments
        MassUIQuoteEncoder.AdjustmentsEncoder adjustments = quoteEncoder.adjustments();
        adjustments.atmConfigurationVersion(2);
        adjustments.adjustmentCategory(AdjustmentCategory.GR2);
        adjustments.additivePipSpread(0.5);
        adjustments.depthOfBookSpreadFactor(0.25);
        adjustments.skew(0.1);
        adjustments.liquidityFactor(1.5);
        adjustments.liquidityCap(100.0);
        adjustments.pricingEnabled(BooleanType.TRUE);
        adjustments.label(LabelType.YELLOW);

        // Stack pricing (optional group with 1 quote)
        MassUIQuoteEncoder.StackPricingEncoder stack = quoteEncoder.stackPricingCount(1);
        stack.next()
             .quoteType(QuoteType.INDICATIVE)
             .adjustmentCategory(AdjustmentCategory.GR1)
             .effectiveTime(1650000000)
             .effectiveTo(1650009999)
             .effectiveDob(0.4);

        MassUIQuoteEncoder.StackPricingEncoder.QuotesEncoder quotes = stack.quotesCount(1);
        quotes.next()
              .volume(1000)
              .bid(1.234)
              .offer(1.236)
              .spread(0.002);

        quoteEncoder.stack("stackData".getBytes(StandardCharsets.UTF_8));
        System.out.println("Message encoded.");
    }

    private static void decode(MutableDirectBuffer buffer) {
        MessageHeaderDecoder headerDecoder = new MessageHeaderDecoder();
        MassUIQuoteDecoder quoteDecoder = new MassUIQuoteDecoder();

        // Wrap with header
        int offset = 0;
        headerDecoder.wrap(buffer, offset);

        quoteDecoder.wrap(
            buffer,
            offset + headerDecoder.encodedLength(),
            headerDecoder.blockLength(),
            headerDecoder.version()
        );

        System.out.println("\nDecoded Quote:");
        System.out.println("Quote ID: " + quoteDecoder.quoteId());
        System.out.println("Sending Time: " + quoteDecoder.sendingTime());

        MassUIQuoteDecoder.CurrencyPairPricingDecoder pricingGroup = quoteDecoder.currencyPairPricing();
        while (pricingGroup.hasNext()) {
            pricingGroup.next();
            System.out.println("Currency Pair: " + pricingGroup.currencyPair());
            System.out.println("PIP: " + pricingGroup.pip());
            System.out.println("Quote Type: " + pricingGroup.quoteType());
        }

        MassUIQuoteDecoder.AdjustmentsDecoder adjustments = quoteDecoder.adjustments();
        System.out.println("Adjustment Category: " + adjustments.adjustmentCategory());
        System.out.println("Pricing Enabled: " + adjustments.pricingEnabled());
        System.out.println("Label: " + adjustments.label());

        MassUIQuoteDecoder.StackPricingDecoder stack = quoteDecoder.stackPricing();
        while (stack.hasNext()) {
            stack.next();
            System.out.println("Stack Quote Type: " + stack.quoteType());
            MassUIQuoteDecoder.StackPricingDecoder.QuotesDecoder quotes = stack.quotes();
            while (quotes.hasNext()) {
                quotes.next();
                System.out.println("Bid: " + quotes.bid() + " Offer: " + quotes.offer());
            }
        }

        byte[] stackData = new byte[quoteDecoder.stackLength()];
        quoteDecoder.getStack(stackData, 0, stackData.length);
        System.out.println("Stack VarData: " + new String(stackData, StandardCharsets.UTF_8));
    }
}

```
