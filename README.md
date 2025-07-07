```java
import lombok.Data;
import java.util.List;

@Data
public class CreditCheckRequest {

    private List<Deal> deals;

    private String product;
    private String baseNumber;
    private String bvShortName;
    private String facilityId;
    private String gfcid;
    private String k1Number;
    private String sourceSystem;
    private String earmarkType;
    private String requestType;
    private String tradeDate;
    private String productDeliveryType;
    private String messageType;

    @Data
    public static class Deal {
        private boolean isNdf;
        private String valueDate;
        private String referenceNumber;
        private String originalReferenceNumber;

        private String boughtCurrency;
        private String soldCurrency;

        private double boughtAmount;
        private double soldAmount;
        private double usDollarAmt;
    }
}

```
