```java
import quickfix.*;
import quickfix.field.*;
import quickfix.fix44.NewOrderSingle;
import java.math.BigDecimal;

/**
 * DTO for FIX New Order Single Message
 */
public class NewOrderSingleDTO {
    private String symbol;
    private String securityType;
    private char side;
    private char timeInForce;
    private String custDomTimeInForce;
    private BigDecimal orderQty;
    private BigDecimal price;
    private String currency;
    private String clOrdID;
    private String clOrdLinkID;
    private String custUserId;
    private String deliverToCompID;
    private String custOriginalExecAuthority;
    private String custFixSessionKey;
    private String custOriginatingTrader;
    private String custOriginatingSession;
    private char ordType;
    private String custEAOrderType;
    private String transactTime;
    private String regulationTradeName;
    private String regulationAlgoFlag;
    private String nosEaNanoTS;
    private String custDropCopy;
    private String custUioLinkID;
    private String custCountryLocation;
    private String custAlgoOrderFlag;
    private String custUioLinkVersion;
    private String custSystemCSIValue;
    private BigDecimal maxShow;
    private BigDecimal displayQTY;
    private String settlDate;
    private String maturityDate;
    private String account;
    private String country;
    private char handlInst;
    private BigDecimal minQty;
    private String mdEntryID;
    private String strategyLinkID;
    private String custUioSysCSIValue;
    private String custLTAShortCode;
    private String ocrNanoTS;

    // Getters and Setters
    public String getSymbol() { return symbol; }
    public void setSymbol(String symbol) { this.symbol = symbol; }

    public String getSecurityType() { return securityType; }
    public void setSecurityType(String securityType) { this.securityType = securityType; }

    public char getSide() { return side; }
    public void setSide(char side) { this.side = side; }

    public char getTimeInForce() { return timeInForce; }
    public void setTimeInForce(char timeInForce) { this.timeInForce = timeInForce; }

    public String getCustDomTimeInForce() { return custDomTimeInForce; }
    public void setCustDomTimeInForce(String custDomTimeInForce) { this.custDomTimeInForce = custDomTimeInForce; }

    public BigDecimal getOrderQty() { return orderQty; }
    public void setOrderQty(BigDecimal orderQty) { this.orderQty = orderQty; }

    public BigDecimal getPrice() { return price; }
    public void setPrice(BigDecimal price) { this.price = price; }

    public String getCurrency() { return currency; }
    public void setCurrency(String currency) { this.currency = currency; }

    public String getClOrdID() { return clOrdID; }
    public void setClOrdID(String clOrdID) { this.clOrdID = clOrdID; }

    public String getClOrdLinkID() { return clOrdLinkID; }
    public void setClOrdLinkID(String clOrdLinkID) { this.clOrdLinkID = clOrdLinkID; }

    public String getCustUserId() { return custUserId; }
    public void setCustUserId(String custUserId) { this.custUserId = custUserId; }

    public String getDeliverToCompID() { return deliverToCompID; }
    public void setDeliverToCompID(String deliverToCompID) { this.deliverToCompID = deliverToCompID; }

    public String getCustOriginalExecAuthority() { return custOriginalExecAuthority; }
    public void setCustOriginalExecAuthority(String custOriginalExecAuthority) { this.custOriginalExecAuthority = custOriginalExecAuthority; }

    public String getCustFixSessionKey() { return custFixSessionKey; }
    public void setCustFixSessionKey(String custFixSessionKey) { this.custFixSessionKey = custFixSessionKey; }

    public String getCustOriginatingTrader() { return custOriginatingTrader; }
    public void setCustOriginatingTrader(String custOriginatingTrader) { this.custOriginatingTrader = custOriginatingTrader; }

    public String getCustOriginatingSession() { return custOriginatingSession; }
    public void setCustOriginatingSession(String custOriginatingSession) { this.custOriginatingSession = custOriginatingSession; }

    public char getOrdType() { return ordType; }
    public void setOrdType(char ordType) { this.ordType = ordType; }

    public String getCustEAOrderType() { return custEAOrderType; }
    public void setCustEAOrderType(String custEAOrderType) { this.custEAOrderType = custEAOrderType; }

    public String getTransactTime() { return transactTime; }
    public void setTransactTime(String transactTime) { this.transactTime = transactTime; }

    public String getRegulationTradeName() { return regulationTradeName; }
    public void setRegulationTradeName(String regulationTradeName) { this.regulationTradeName = regulationTradeName; }

    public String getRegulationAlgoFlag() { return regulationAlgoFlag; }
    public void setRegulationAlgoFlag(String regulationAlgoFlag) { this.regulationAlgoFlag = regulationAlgoFlag; }

    public String getNosEaNanoTS() { return nosEaNanoTS; }
    public void setNosEaNanoTS(String nosEaNanoTS) { this.nosEaNanoTS = nosEaNanoTS; }

    public String getCustDropCopy() { return custDropCopy; }
    public void setCustDropCopy(String custDropCopy) { this.custDropCopy = custDropCopy; }

    public String getCustUioLinkID() { return custUioLinkID; }
    public void setCustUioLinkID(String custUioLinkID) { this.custUioLinkID = custUioLinkID; }

    public String getCustCountryLocation() { return custCountryLocation; }
    public void setCustCountryLocation(String custCountryLocation) { this.custCountryLocation = custCountryLocation; }

    public String getCustAlgoOrderFlag() { return custAlgoOrderFlag; }
    public void setCustAlgoOrderFlag(String custAlgoOrderFlag) { this.custAlgoOrderFlag = custAlgoOrderFlag; }

    public String getCustUioLinkVersion() { return custUioLinkVersion; }
    public void setCustUioLinkVersion(String custUioLinkVersion) { this.custUioLinkVersion = custUioLinkVersion; }

    public String getCustSystemCSIValue() { return custSystemCSIValue; }
    public void setCustSystemCSIValue(String custSystemCSIValue) { this.custSystemCSIValue = custSystemCSIValue; }

    public BigDecimal getMaxShow() { return maxShow; }
    public void setMaxShow(BigDecimal maxShow) { this.maxShow = maxShow; }

    public BigDecimal getDisplayQTY() { return displayQTY; }
    public void setDisplayQTY(BigDecimal displayQTY) { this.displayQTY = displayQTY; }

    public String getSettlDate() { return settlDate; }
    public void setSettlDate(String settlDate) { this.settlDate = settlDate; }

    public String getMaturityDate() { return maturityDate; }
    public void setMaturityDate(String maturityDate) { this.maturityDate = maturityDate; }

    public String getAccount() { return account; }
    public void setAccount(String account) { this.account = account; }

    public String getCountry() { return country; }
    public void setCountry(String country) { this.country = country; }

    public char getHandlInst() { return handlInst; }
    public void setHandlInst(char handlInst) { this.handlInst = handlInst; }

    public BigDecimal getMinQty() { return minQty; }
    public void setMinQty(BigDecimal minQty) { this.minQty = minQty; }

    public String getMdEntryID() { return mdEntryID; }
    public void setMdEntryID(String mdEntryID) { this.mdEntryID = mdEntryID; }

    public String getStrategyLinkID() { return strategyLinkID; }
    public void setStrategyLinkID(String strategyLinkID) { this.strategyLinkID = strategyLinkID; }

    public String getCustUioSysCSIValue() { return custUioSysCSIValue; }
    public void setCustUioSysCSIValue(String custUioSysCSIValue) { this.custUioSysCSIValue = custUioSysCSIValue; }

    public String getCustLTAShortCode() { return custLTAShortCode; }
    public void setCustLTAShortCode(String custLTAShortCode) { this.custLTAShortCode = custLTAShortCode; }

    public String getOcrNanoTS() { return ocrNanoTS; }
    public void setOcrNanoTS(String ocrNanoTS) { this.ocrNanoTS = ocrNanoTS; }

    // Builder pattern
    public static class Builder {
        private NewOrderSingleDTO dto = new NewOrderSingleDTO();

        public Builder symbol(String symbol) { dto.symbol = symbol; return this; }
        public Builder securityType(String securityType) { dto.securityType = securityType; return this; }
        public Builder side(char side) { dto.side = side; return this; }
        public Builder timeInForce(char timeInForce) { dto.timeInForce = timeInForce; return this; }
        public Builder custDomTimeInForce(String custDomTimeInForce) { dto.custDomTimeInForce = custDomTimeInForce; return this; }
        public Builder orderQty(BigDecimal orderQty) { dto.orderQty = orderQty; return this; }
        public Builder price(BigDecimal price) { dto.price = price; return this; }
        public Builder currency(String currency) { dto.currency = currency; return this; }
        public Builder clOrdID(String clOrdID) { dto.clOrdID = clOrdID; return this; }
        public Builder clOrdLinkID(String clOrdLinkID) { dto.clOrdLinkID = clOrdLinkID; return this; }
        public Builder custUserId(String custUserId) { dto.custUserId = custUserId; return this; }
        public Builder deliverToCompID(String deliverToCompID) { dto.deliverToCompID = deliverToCompID; return this; }
        public Builder custOriginalExecAuthority(String custOriginalExecAuthority) { dto.custOriginalExecAuthority = custOriginalExecAuthority; return this; }
        public Builder custFixSessionKey(String custFixSessionKey) { dto.custFixSessionKey = custFixSessionKey; return this; }
        public Builder custOriginatingTrader(String custOriginatingTrader) { dto.custOriginatingTrader = custOriginatingTrader; return this; }
        public Builder custOriginatingSession(String custOriginatingSession) { dto.custOriginatingSession = custOriginatingSession; return this; }
        public Builder ordType(char ordType) { dto.ordType = ordType; return this; }
        public Builder custEAOrderType(String custEAOrderType) { dto.custEAOrderType = custEAOrderType; return this; }
        public Builder transactTime(String transactTime) { dto.transactTime = transactTime; return this; }
        public Builder regulationTradeName(String regulationTradeName) { dto.regulationTradeName = regulationTradeName; return this; }
        public Builder regulationAlgoFlag(String regulationAlgoFlag) { dto.regulationAlgoFlag = regulationAlgoFlag; return this; }
        public Builder nosEaNanoTS(String nosEaNanoTS) { dto.nosEaNanoTS = nosEaNanoTS; return this; }
        public Builder custDropCopy(String custDropCopy) { dto.custDropCopy = custDropCopy; return this; }
        public Builder custUioLinkID(String custUioLinkID) { dto.custUioLinkID = custUioLinkID; return this; }
        public Builder custCountryLocation(String custCountryLocation) { dto.custCountryLocation = custCountryLocation; return this; }
        public Builder custAlgoOrderFlag(String custAlgoOrderFlag) { dto.custAlgoOrderFlag = custAlgoOrderFlag; return this; }
        public Builder custUioLinkVersion(String custUioLinkVersion) { dto.custUioLinkVersion = custUioLinkVersion; return this; }
        public Builder custSystemCSIValue(String custSystemCSIValue) { dto.custSystemCSIValue = custSystemCSIValue; return this; }
        public Builder maxShow(BigDecimal maxShow) { dto.maxShow = maxShow; return this; }
        public Builder displayQTY(BigDecimal displayQTY) { dto.displayQTY = displayQTY; return this; }
        public Builder settlDate(String settlDate) { dto.settlDate = settlDate; return this; }
        public Builder maturityDate(String maturityDate) { dto.maturityDate = maturityDate; return this; }
        public Builder account(String account) { dto.account = account; return this; }
        public Builder country(String country) { dto.country = country; return this; }
        public Builder handlInst(char handlInst) { dto.handlInst = handlInst; return this; }
        public Builder minQty(BigDecimal minQty) { dto.minQty = minQty; return this; }
        public Builder mdEntryID(String mdEntryID) { dto.mdEntryID = mdEntryID; return this; }
        public Builder strategyLinkID(String strategyLinkID) { dto.strategyLinkID = strategyLinkID; return this; }
        public Builder custUioSysCSIValue(String custUioSysCSIValue) { dto.custUioSysCSIValue = custUioSysCSIValue; return this; }
        public Builder custLTAShortCode(String custLTAShortCode) { dto.custLTAShortCode = custLTAShortCode; return this; }
        public Builder ocrNanoTS(String ocrNanoTS) { dto.ocrNanoTS = ocrNanoTS; return this; }

        public NewOrderSingleDTO build() { return dto; }
    }
}

/**
 * Custom FIX Field Definitions
 */
class CustDomTimeInForce extends StringField {
    static final int FIELD = 5001;
    public CustDomTimeInForce() { super(FIELD); }
    public CustDomTimeInForce(String data) { super(FIELD, data); }
}

class CustUserId extends StringField {
    static final int FIELD = 5002;
    public CustUserId() { super(FIELD); }
    public CustUserId(String data) { super(FIELD, data); }
}

class DeliverToCompID extends StringField {
    static final int FIELD = 5003;
    public DeliverToCompID() { super(FIELD); }
    public DeliverToCompID(String data) { super(FIELD, data); }
}

class CustOriginalExecAuthority extends StringField {
    static final int FIELD = 5004;
    public CustOriginalExecAuthority() { super(FIELD); }
    public CustOriginalExecAuthority(String data) { super(FIELD, data); }
}

class CustFixSessionKey extends StringField {
    static final int FIELD = 5005;
    public CustFixSessionKey() { super(FIELD); }
    public CustFixSessionKey(String data) { super(FIELD, data); }
}

class CustOriginatingTrader extends StringField {
    static final int FIELD = 5006;
    public CustOriginatingTrader() { super(FIELD); }
    public CustOriginatingTrader(String data) { super(FIELD, data); }
}

class CustOriginatingSession extends StringField {
    static final int FIELD = 5007;
    public CustOriginatingSession() { super(FIELD); }
    public CustOriginatingSession(String data) { super(FIELD, data); }
}

class CustEAOrderType extends StringField {
    static final int FIELD = 5008;
    public CustEAOrderType() { super(FIELD); }
    public CustEAOrderType(String data) { super(FIELD, data); }
}

class RegulationTradeName extends StringField {
    static final int FIELD = 5009;
    public RegulationTradeName() { super(FIELD); }
    public RegulationTradeName(String data) { super(FIELD, data); }
}

class RegulationAlgoFlag extends StringField {
    static final int FIELD = 5010;
    public RegulationAlgoFlag() { super(FIELD); }
    public RegulationAlgoFlag(String data) { super(FIELD, data); }
}

class NosEaNanoTS extends StringField {
    static final int FIELD = 5011;
    public NosEaNanoTS() { super(FIELD); }
    public NosEaNanoTS(String data) { super(FIELD, data); }
}

class CustDropCopy extends StringField {
    static final int FIELD = 5012;
    public CustDropCopy() { super(FIELD); }
    public CustDropCopy(String data) { super(FIELD, data); }
}

class CustUioLinkID extends StringField {
    static final int FIELD = 5013;
    public CustUioLinkID() { super(FIELD); }
    public CustUioLinkID(String data) { super(FIELD, data); }
}

class CustCountryLocation extends StringField {
    static final int FIELD = 5014;
    public CustCountryLocation() { super(FIELD); }
    public CustCountryLocation(String data) { super(FIELD, data); }
}

class CustAlgoOrderFlag extends StringField {
    static final int FIELD = 5015;
    public CustAlgoOrderFlag() { super(FIELD); }
    public CustAlgoOrderFlag(String data) { super(FIELD, data); }
}

class CustUioLinkVersion extends StringField {
    static final int FIELD = 5016;
    public CustUioLinkVersion() { super(FIELD); }
    public CustUioLinkVersion(String data) { super(FIELD, data); }
}

class CustSystemCSIValue extends StringField {
    static final int FIELD = 5017;
    public CustSystemCSIValue() { super(FIELD); }
    public CustSystemCSIValue(String data) { super(FIELD, data); }
}

class DisplayQTY extends DecimalField {
    static final int FIELD = 5018;
    public DisplayQTY() { super(FIELD); }
    public DisplayQTY(BigDecimal data) { super(FIELD, data); }
}

class CustUioSysCSIValue extends StringField {
    static final int FIELD = 5019;
    public CustUioSysCSIValue() { super(FIELD); }
    public CustUioSysCSIValue(String data) { super(FIELD, data); }
}

class CustLTAShortCode extends StringField {
    static final int FIELD = 5020;
    public CustLTAShortCode() { super(FIELD); }
    public CustLTAShortCode(String data) { super(FIELD, data); }
}

class OcrNanoTS extends StringField {
    static final int FIELD = 5021;
    public OcrNanoTS() { super(FIELD); }
    public OcrNanoTS(String data) { super(FIELD, data); }
}

/**
 * FIX Message Factory using QuickFIX/J for creating proper New Order Single messages
 */
class FixMessageFactory {

    /**
     * Creates a proper QuickFIX/J NewOrderSingle message from a DTO
     */
    public static NewOrderSingle createNewOrderSingle(NewOrderSingleDTO dto) throws FieldNotFound {
        NewOrderSingle message = new NewOrderSingle();
        
        // Required fields
        if (dto.getClOrdID() != null) {
            message.set(new ClOrdID(dto.getClOrdID()));
        }
        
        if (dto.getSide() != '\0') {
            message.set(new Side(dto.getSide()));
        }
        
        if (dto.getTransactTime() != null) {
            message.set(new TransactTime());
        }
        
        if (dto.getOrdType() != '\0') {
            message.set(new OrdType(dto.getOrdType()));
        }
        
        // Symbol
        if (dto.getSymbol() != null) {
            message.set(new Symbol(dto.getSymbol()));
        }
        
        // SecurityType
        if (dto.getSecurityType() != null) {
            message.set(new SecurityType(dto.getSecurityType()));
        }
        
        // TimeInForce
        if (dto.getTimeInForce() != '\0') {
            message.set(new TimeInForce(dto.getTimeInForce()));
        }
        
        // OrderQty
        if (dto.getOrderQty() != null) {
            message.set(new OrderQty(dto.getOrderQty().doubleValue()));
        }
        
        // Price
        if (dto.getPrice() != null) {
            message.set(new Price(dto.getPrice().doubleValue()));
        }
        
        // Currency
        if (dto.getCurrency() != null) {
            message.set(new Currency(dto.getCurrency()));
        }
        
        // ClOrdLinkID
        if (dto.getClOrdLinkID() != null) {
            message.set(new ClOrdLinkID(dto.getClOrdLinkID()));
        }
        
        // Account
        if (dto.getAccount() != null) {
            message.set(new Account(dto.getAccount()));
        }
        
        // Country
        if (dto.getCountry() != null) {
            message.set(new Country(dto.getCountry()));
        }
        
        // HandlInst
        if (dto.getHandlInst() != '\0') {
            message.set(new HandlInst(dto.getHandlInst()));
        }
        
        // MinQty
        if (dto.getMinQty() != null) {
            message.set(new MinQty(dto.getMinQty().doubleValue()));
        }
        
        // MaxShow
        if (dto.getMaxShow() != null) {
            message.set(new MaxShow(dto.getMaxShow().doubleValue()));
        }
        
        // SettlDate
        if (dto.getSettlDate() != null) {
            message.set(new SettlDate(dto.getSettlDate()));
        }
        
        // MaturityDate
        if (dto.getMaturityDate() != null) {
            message.set(new MaturityDate(dto.getMaturityDate()));
        }
        
        // MDEntryID
        if (dto.getMdEntryID() != null) {
            message.set(new MDEntryID(dto.getMdEntryID()));
        }
        
        // StrategyLinkID  
        if (dto.getStrategyLinkID() != null) {
            message.set(new CustUioLinkID(dto.getStrategyLinkID()));
        }
        
        // Custom fields using typed field classes
        if (dto.getCustDomTimeInForce() != null) {
            message.set(new CustDomTimeInForce(dto.getCustDomTimeInForce()));
        }
        if (dto.getCustUserId() != null) {
            message.set(new CustUserId(dto.getCustUserId()));
        }
        if (dto.getDeliverToCompID() != null) {
            message.set(new DeliverToCompID(dto.getDeliverToCompID()));
        }
        if (dto.getCustOriginalExecAuthority() != null) {
            message.set(new CustOriginalExecAuthority(dto.getCustOriginalExecAuthority()));
        }
        if (dto.getCustFixSessionKey() != null) {
            message.set(new CustFixSessionKey(dto.getCustFixSessionKey()));
        }
        if (dto.getCustOriginatingTrader() != null) {
            message.set(new CustOriginatingTrader(dto.getCustOriginatingTrader()));
        }
        if (dto.getCustOriginatingSession() != null) {
            message.set(new CustOriginatingSession(dto.getCustOriginatingSession()));
        }
        if (dto.getCustEAOrderType() != null) {
            message.set(new CustEAOrderType(dto.getCustEAOrderType()));
        }
        if (dto.getRegulationTradeName() != null) {
            message.set(new RegulationTradeName(dto.getRegulationTradeName()));
        }
        if (dto.getRegulationAlgoFlag() != null) {
            message.set(new RegulationAlgoFlag(dto.getRegulationAlgoFlag()));
        }
        if (dto.getNosEaNanoTS() != null) {
            message.set(new NosEaNanoTS(dto.getNosEaNanoTS()));
        }
        if (dto.getCustDropCopy() != null) {
            message.set(new CustDropCopy(dto.getCustDropCopy()));
        }
        if (dto.getCustCountryLocation() != null) {
            message.set(new CustCountryLocation(dto.getCustCountryLocation()));
        }
        if (dto.getCustAlgoOrderFlag() != null) {
            message.set(new CustAlgoOrderFlag(dto.getCustAlgoOrderFlag()));
        }
        if (dto.getCustUioLinkVersion() != null) {
            message.set(new CustUioLinkVersion(dto.getCustUioLinkVersion()));
        }
        if (dto.getCustSystemCSIValue() != null) {
            message.set(new CustSystemCSIValue(dto.getCustSystemCSIValue()));
        }
        if (dto.getCustUioSysCSIValue() != null) {
            message.set(new CustUioSysCSIValue(dto.getCustUioSysCSIValue()));
        }
        if (dto.getCustLTAShortCode() != null) {
            message.set(new CustLTAShortCode(dto.getCustLTAShortCode()));
        }
        if (dto.getOcrNanoTS() != null) {
            message.set(new OcrNanoTS(dto.getOcrNanoTS()));
        }
        if (dto.getDisplayQTY() != null) {
            message.set(new DisplayQTY(dto.getDisplayQTY()));
        }
        
        return message;
    }

    /**
     * Example usage demonstrating how to create and send a FIX message
     */
    public static void main(String[] args) {
        try {
            // Build a DTO using the builder pattern
            NewOrderSingleDTO order = new NewOrderSingleDTO.Builder()
                .symbol("AAPL")
                .side(Side.BUY)
                .orderQty(new BigDecimal("100"))
                .price(new BigDecimal("150.25"))
                .ordType(OrdType.LIMIT)
                .timeInForce(TimeInForce.DAY)
                .clOrdID("ORDER123")
                .currency("USD")
                .account("ACC001")
                .custUserId("TRADER01")
                .custAlgoOrderFlag("Y")
                .build();

            // Create proper FIX message
            NewOrderSingle fixMessage = FixMessageFactory.createNewOrderSingle(order);
            
            // The fixMessage object can now be sent via QuickFIX/J Session
            // Session.sendToTarget(fixMessage, sessionID);
            
            // Or convert to string for logging
            System.out.println("FIX Message created:");
            System.out.println(fixMessage.toString());
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```
