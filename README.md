```java

package com.citi.fxpg.model;

import jakarta.xml.bind.annotation.*;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import java.util.List;

@Data
@NoArgsConstructor
@AllArgsConstructor
@XmlRootElement(name = "TradeBookingRequest_3.0")
@XmlAccessorType(XmlAccessType.FIELD)
public class TradeBookingRequest {
    
    @XmlElement(name = "SystemDetails")
    private SystemDetails systemDetails;
    
    @XmlElement(name = "TradeDetails")
    private TradeDetails tradeDetails;
    
    @XmlElement(name = "INPUT_TIME")
    private String inputTime;
    
    @XmlElement(name = "TradeAttributeList")
    private TradeAttributeList tradeAttributeList;
}

@Data
@NoArgsConstructor
@AllArgsConstructor
@XmlAccessorType(XmlAccessType.FIELD)
class SystemDetails {
    
    @XmlElement(name = "CLIENT_SYSTEM")
    private String clientSystem;
    
    @XmlElement(name = "CLIENT_SUBSYSTEM")
    private String clientSubsystem;
    
    @XmlElement(name = "TRANSMITTED_DATETIME")
    private String transmittedDateTime;
}

@Data
@NoArgsConstructor
@AllArgsConstructor
@XmlAccessorType(XmlAccessType.FIELD)
class TradeDetails {
    
    @XmlElement(name = "CLIENT_REFERENCE")
    private String clientReference;
    
    @XmlElement(name = "BOOKING_BRANCH")
    private String bookingBranch;
    
    @XmlElement(name = "CUSTOMER_DEALER_ID")
    private String customerDealerId;
    
    @XmlElement(name = "CCY1_DEALER_ID")
    private String ccy1DealerId;
    
    @XmlElement(name = "CCY2_DEALER_ID")
    private String ccy2DealerId;
    
    @XmlElement(name = "BROKER_CODE")
    private String brokerCode;
    
    @XmlElement(name = "IS_CUSTOMER_TRADE")
    private String isCustomerTrade;
    
    @XmlElement(name = "CUSTOMERID")
    private String customerId;
    
    @XmlElement(name = "CUSTOMER_NAME")
    private String customerName;
    
    @XmlElement(name = "CUSTOMER_TYPE")
    private String customerType;
    
    @XmlElement(name = "TRADE_INSTRUMENT")
    private String tradeInstrument;
    
    @XmlElement(name = "TRADE_ACTION")
    private String tradeAction;
    
    @XmlElement(name = "CROSS_RATE_TYPE")
    private String crossRateType;
    
    @XmlElement(name = "MARKET_FIELD")
    private String marketField;
    
    @XmlElement(name = "BOOKING_DATE")
    private String bookingDate;
    
    @XmlElement(name = "TRADE_LEGS")
    private TradeLegs tradeLegs;
}

@Data
@NoArgsConstructor
@AllArgsConstructor
@XmlAccessorType(XmlAccessType.FIELD)
class TradeLegs {
    
    @XmlElement(name = "TradeLeg")
    private List<TradeLeg> tradeLegs;
}

@Data
@NoArgsConstructor
@AllArgsConstructor
@XmlAccessorType(XmlAccessType.FIELD)
class TradeLeg {
    
    @XmlElement(name = "LEG_NO")
    private String legNo;
    
    @XmlElement(name = "LEG_TYPE")
    private String legType;
    
    @XmlElement(name = "SPOT_DATE")
    private String spotDate;
    
    @XmlElement(name = "CCY1_VALUE_DATE")
    private String ccy1ValueDate;
    
    @XmlElement(name = "CCY2_VALUE_DATE")
    private String ccy2ValueDate;
    
    @XmlElement(name = "CCY1_BSIND")
    private String ccy1BsInd;
    
    @XmlElement(name = "CURRENCY1")
    private String currency1;
    
    @XmlElement(name = "CURRENCY2")
    private String currency2;
    
    @XmlElement(name = "CCY1_CCY2_BASE")
    private String ccy1Ccy2Base;
    
    @XmlElement(name = "AMOUNT1")
    private String amount1;
    
    @XmlElement(name = "AMOUNT2")
    private String amount2;
    
    @XmlElement(name = "SPOT_DEAL_RATE")
    private String spotDealRate;
    
    @XmlElement(name = "SPOT_COVER_RATE")
    private String spotCoverRate;
}

@Data
@NoArgsConstructor
@AllArgsConstructor
@XmlAccessorType(XmlAccessType.FIELD)
class TradeAttributeList {
    
    @XmlElement(name = "TradeAttributeInfo")
    private List<TradeAttributeInfo> tradeAttributeInfo;
}

@Data
@NoArgsConstructor
@AllArgsConstructor
@XmlAccessorType(XmlAccessType.FIELD)
class TradeAttributeInfo {
    
    @XmlAttribute(name = "name")
    private String name;
    
    @XmlAttribute(name = "value")
    private String value;
}

```
