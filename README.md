```java


// ConnectionDetails.java
public class ConnectionDetails {
    private final String host;
    private final int port;
    private final String senderCompId;
    private final String targetCompId;
    private final String username;
    private final String password;
    private final String fixVersion;
    private final int heartbeatInterval;
    private final boolean resetOnLogon;
    private final String logFileDirectory;
    private final String storeFileDirectory;

    private ConnectionDetails(Builder builder) {
        this.host = builder.host;
        this.port = builder.port;
        this.senderCompId = builder.senderCompId;
        this.targetCompId = builder.targetCompId;
        this.username = builder.username;
        this.password = builder.password;
        this.fixVersion = builder.fixVersion;
        this.heartbeatInterval = builder.heartbeatInterval;
        this.resetOnLogon = builder.resetOnLogon;
        this.logFileDirectory = builder.logFileDirectory;
        this.storeFileDirectory = builder.storeFileDirectory;
    }

    // Getters
    public String getHost() { return host; }
    public int getPort() { return port; }
    public String getSenderCompId() { return senderCompId; }
    public String getTargetCompId() { return targetCompId; }
    public String getUsername() { return username; }
    public String getPassword() { return password; }
    public String getFixVersion() { return fixVersion; }
    public int getHeartbeatInterval() { return heartbeatInterval; }
    public boolean isResetOnLogon() { return resetOnLogon; }
    public String getLogFileDirectory() { return logFileDirectory; }
    public String getStoreFileDirectory() { return storeFileDirectory; }

    public static class Builder {
        private String host;
        private int port;
        private String senderCompId;
        private String targetCompId;
        private String username;
        private String password;
        private String fixVersion = "FIX.4.4";
        private int heartbeatInterval = 30;
        private boolean resetOnLogon = false;
        private String logFileDirectory = "./logs";
        private String storeFileDirectory = "./store";

        public Builder host(String host) { this.host = host; return this; }
        public Builder port(int port) { this.port = port; return this; }
        public Builder senderCompId(String senderCompId) { this.senderCompId = senderCompId; return this; }
        public Builder targetCompId(String targetCompId) { this.targetCompId = targetCompId; return this; }
        public Builder username(String username) { this.username = username; return this; }
        public Builder password(String password) { this.password = password; return this; }
        public Builder fixVersion(String fixVersion) { this.fixVersion = fixVersion; return this; }
        public Builder heartbeatInterval(int heartbeatInterval) { this.heartbeatInterval = heartbeatInterval; return this; }
        public Builder resetOnLogon(boolean resetOnLogon) { this.resetOnLogon = resetOnLogon; return this; }
        public Builder logFileDirectory(String logFileDirectory) { this.logFileDirectory = logFileDirectory; return this; }
        public Builder storeFileDirectory(String storeFileDirectory) { this.storeFileDirectory = storeFileDirectory; return this; }

        public ConnectionDetails build() {
            if (host == null || senderCompId == null || targetCompId == null) {
                throw new IllegalArgumentException("Host, SenderCompId, and TargetCompId are required");
            }
            return new ConnectionDetails(this);
        }
    }
}

// SessionType.java
public enum SessionType {
    TRADING("TRADING"),
    QUOTE("QUOTE");

    private final String value;

    SessionType(String value) {
        this.value = value;
    }

    public String getValue() {
        return value;
    }
}

// QuickFixClientException.java
public class QuickFixClientException extends Exception {
    public QuickFixClientException(String message) {
        super(message);
    }

    public QuickFixClientException(String message, Throwable cause) {
        super(message, cause);
    }
}

// SessionEventListener.java
import quickfix.SessionID;

public interface SessionEventListener {
    void onLogon(SessionID sessionId);
    void onLogout(SessionID sessionId);
    void onReject(SessionID sessionId, String reason);
    void onError(SessionID sessionId, String error);
}

// MessageListener.java
import quickfix.Message;
import quickfix.SessionID;

public interface MessageListener {
    void onMessage(SessionID sessionId, Message message);
}

// QuickFixClient.java
import quickfix.*;
import quickfix.field.*;
import quickfix.fix44.MessageFactory;
import quickfix.fix44.NewOrderSingle;
import quickfix.fix44.MarketDataRequest;
import quickfix.fix44.QuoteRequest;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.File;
import java.io.FileWriter;
import java.io.IOException;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.ConcurrentHashMap;
import java.util.Map;

public class QuickFixClient implements Application {
    private static final Logger logger = LoggerFactory.getLogger(QuickFixClient.class);
    
    private final ConnectionDetails connectionDetails;
    private final SessionType sessionType;
    private final AtomicBoolean connected = new AtomicBoolean(false);
    private final CountDownLatch logonLatch = new CountDownLatch(1);
    
    private SocketInitiator initiator;
    private SessionID sessionId;
    private SessionSettings settings;
    
    private MessageListener messageListener;
    private SessionEventListener sessionEventListener;
    
    private final Map<String, String> customFields = new ConcurrentHashMap<>();

    public QuickFixClient(ConnectionDetails connectionDetails, SessionType sessionType) {
        this.connectionDetails = connectionDetails;
        this.sessionType = sessionType;
        this.settings = createSessionSettings();
    }

    public void setMessageListener(MessageListener listener) {
        this.messageListener = listener;
    }

    public void setSessionEventListener(SessionEventListener listener) {
        this.sessionEventListener = listener;
    }

    public void addCustomField(String key, String value) {
        customFields.put(key, value);
    }

    public void connect() throws QuickFixClientException {
        try {
            logger.info("Connecting to {}:{} with session type {}", 
                       connectionDetails.getHost(), connectionDetails.getPort(), sessionType);
            
            createDirectories();
            
            MessageStoreFactory storeFactory = new FileStoreFactory(settings);
            LogFactory logFactory = new FileLogFactory(settings);
            MessageFactory messageFactory = new MessageFactory();
            
            initiator = new SocketInitiator(this, storeFactory, settings, logFactory, messageFactory);
            initiator.start();
            
            // Wait for logon with timeout
            if (!logonLatch.await(30, TimeUnit.SECONDS)) {
                throw new QuickFixClientException("Connection timeout - failed to logon within 30 seconds");
            }
            
            logger.info("Successfully connected and logged on");
            
        } catch (ConfigError | InterruptedException e) {
            throw new QuickFixClientException("Failed to connect", e);
        }
    }

    public void disconnect() {
        try {
            if (initiator != null) {
                logger.info("Disconnecting from session");
                connected.set(false);
                initiator.stop();
                logger.info("Disconnected successfully");
            }
        } catch (Exception e) {
            logger.error("Error during disconnect", e);
        }
    }

    public boolean isConnected() {
        return connected.get();
    }

    // Trading session methods
    public void sendNewOrderSingle(String symbol, char side, double quantity, double price, 
                                 char ordType, String clOrdId) throws QuickFixClientException {
        if (sessionType != SessionType.TRADING) {
            throw new QuickFixClientException("New Order Single can only be sent in TRADING session");
        }
        
        try {
            NewOrderSingle order = new NewOrderSingle();
            order.set(new ClOrdID(clOrdId));
            order.set(new Symbol(symbol));
            order.set(new Side(side));
            order.set(new TransactTime());
            order.set(new OrdType(ordType));
            order.set(new OrderQty(quantity));
            
            if (ordType == OrdType.LIMIT) {
                order.set(new Price(price));
            }
            
            // Add custom fields if any
            addCustomFieldsToMessage(order);
            
            Session.sendToTarget(order, sessionId);
            logger.info("Sent New Order Single: ClOrdID={}, Symbol={}, Side={}, Qty={}", 
                       clOrdId, symbol, side, quantity);
                       
        } catch (SessionNotFound e) {
            throw new QuickFixClientException("Session not found when sending order", e);
        }
    }

    // Quote session methods
    public void sendQuoteRequest(String quoteReqId, String symbol) throws QuickFixClientException {
        if (sessionType != SessionType.QUOTE) {
            throw new QuickFixClientException("Quote Request can only be sent in QUOTE session");
        }
        
        try {
            QuoteRequest quoteRequest = new QuoteRequest();
            quoteRequest.set(new QuoteReqID(quoteReqId));
            
            QuoteRequest.NoRelatedSym noRelatedSym = new QuoteRequest.NoRelatedSym();
            noRelatedSym.set(new Symbol(symbol));
            quoteRequest.addGroup(noRelatedSym);
            
            // Add custom fields if any
            addCustomFieldsToMessage(quoteRequest);
            
            Session.sendToTarget(quoteRequest, sessionId);
            logger.info("Sent Quote Request: QuoteReqID={}, Symbol={}", quoteReqId, symbol);
            
        } catch (SessionNotFound e) {
            throw new QuickFixClientException("Session not found when sending quote request", e);
        }
    }

    public void sendMarketDataRequest(String mdReqId, char subscriptionRequestType, 
                                    int marketDepth, String... symbols) throws QuickFixClientException {
        try {
            MarketDataRequest request = new MarketDataRequest();
            request.set(new MDReqID(mdReqId));
            request.set(new SubscriptionRequestType(subscriptionRequestType));
            request.set(new MarketDepth(marketDepth));
            
            // Add symbols
            for (String symbol : symbols) {
                MarketDataRequest.NoRelatedSym noRelatedSym = new MarketDataRequest.NoRelatedSym();
                noRelatedSym.set(new Symbol(symbol));
                request.addGroup(noRelatedSym);
            }
            
            // Add entry types (bid/offer)
            MarketDataRequest.NoMDEntryTypes bidEntry = new MarketDataRequest.NoMDEntryTypes();
            bidEntry.set(new MDEntryType(MDEntryType.BID));
            request.addGroup(bidEntry);
            
            MarketDataRequest.NoMDEntryTypes offerEntry = new MarketDataRequest.NoMDEntryTypes();
            offerEntry.set(new MDEntryType(MDEntryType.OFFER));
            request.addGroup(offerEntry);
            
            // Add custom fields if any
            addCustomFieldsToMessage(request);
            
            Session.sendToTarget(request, sessionId);
            logger.info("Sent Market Data Request: MDReqID={}, Symbols={}", mdReqId, String.join(",", symbols));
            
        } catch (SessionNotFound e) {
            throw new QuickFixClientException("Session not found when sending market data request", e);
        }
    }

    // Application interface methods
    @Override
    public void onCreate(SessionID sessionId) {
        logger.info("Session created: {}", sessionId);
        this.sessionId = sessionId;
    }

    @Override
    public void onLogon(SessionID sessionId) {
        logger.info("Logged on: {}", sessionId);
        connected.set(true);
        logonLatch.countDown();
        
        if (sessionEventListener != null) {
            sessionEventListener.onLogon(sessionId);
        }
    }

    @Override
    public void onLogout(SessionID sessionId) {
        logger.info("Logged out: {}", sessionId);
        connected.set(false);
        
        if (sessionEventListener != null) {
            sessionEventListener.onLogout(sessionId);
        }
    }

    @Override
    public void toAdmin(Message message, SessionID sessionId) throws DoNotSend {
        try {
            // Add username/password for logon message
            if (message instanceof quickfix.fix44.Logon) {
                if (connectionDetails.getUsername() != null && connectionDetails.getPassword() != null) {
                    message.setString(Username.FIELD, connectionDetails.getUsername());
                    message.setString(Password.FIELD, connectionDetails.getPassword());
                }
                
                // Add reset sequence flag
                if (connectionDetails.isResetOnLogon()) {
                    message.setBoolean(ResetSeqNumFlag.FIELD, true);
                }
            }
            
            logger.debug("Sending admin message: {}", message);
            
        } catch (FieldNotFound e) {
            logger.error("Field not found in admin message", e);
        }
    }

    @Override
    public void fromAdmin(Message message, SessionID sessionId) throws FieldNotFound, 
                                                                      IncorrectDataFormat, 
                                                                      IncorrectTagValue, 
                                                                      RejectLogon {
        logger.debug("Received admin message: {}", message);
        
        if (message instanceof quickfix.fix44.Reject) {
            String reason = "Unknown";
            if (message.isSetField(Text.FIELD)) {
                reason = message.getString(Text.FIELD);
            }
            logger.warn("Received reject message: {}", reason);
            
            if (sessionEventListener != null) {
                sessionEventListener.onReject(sessionId, reason);
            }
        }
    }

    @Override
    public void toApp(Message message, SessionID sessionId) throws DoNotSend {
        logger.debug("Sending application message: {}", message);
    }

    @Override
    public void fromApp(Message message, SessionID sessionId) throws FieldNotFound, 
                                                                     IncorrectDataFormat, 
                                                                     IncorrectTagValue, 
                                                                     UnsupportedMessageType {
        logger.debug("Received application message: {}", message);
        
        if (messageListener != null) {
            messageListener.onMessage(sessionId, message);
        }
    }

    private SessionSettings createSessionSettings() {
        try {
            SessionSettings settings = new SessionSettings();
            SessionID sessionId = new SessionID(connectionDetails.getFixVersion(), 
                                               connectionDetails.getSenderCompId(), 
                                               connectionDetails.getTargetCompId());
            
            Dictionary dict = new Dictionary();
            dict.setString("ConnectionType", "initiator");
            dict.setString("SocketConnectHost", connectionDetails.getHost());
            dict.setString("SocketConnectPort", String.valueOf(connectionDetails.getPort()));
            dict.setString("HeartBtInt", String.valueOf(connectionDetails.getHeartbeatInterval()));
            dict.setString("StartTime", "00:00:00");
            dict.setString("EndTime", "00:00:00");
            dict.setString("UseDataDictionary", "Y");
            dict.setString("ValidateUserDefinedFields", "N");
            dict.setString("ValidateIncomingMessage", "N");
            dict.setString("ValidateFieldsOutOfOrder", "N");
            dict.setString("LogonTimeout", "30");
            dict.setString("LogoutTimeout", "5");
            
            // File paths
            dict.setString("FileStorePath", connectionDetails.getStoreFileDirectory());
            dict.setString("FileLogPath", connectionDetails.getLogFileDirectory());
            
            // Session type specific settings
            if (sessionType == SessionType.QUOTE) {
                dict.setString("EnableNextExpectedMsgSeqNum", "N");
            }
            
            settings.set(sessionId, dict);
            
            return settings;
            
        } catch (ConfigError e) {
            throw new RuntimeException("Failed to create session settings", e);
        }
    }

    private void createDirectories() throws QuickFixClientException {
        try {
            File logDir = new File(connectionDetails.getLogFileDirectory());
            File storeDir = new File(connectionDetails.getStoreFileDirectory());
            
            if (!logDir.exists() && !logDir.mkdirs()) {
                throw new QuickFixClientException("Failed to create log directory: " + logDir.getAbsolutePath());
            }
            
            if (!storeDir.exists() && !storeDir.mkdirs()) {
                throw new QuickFixClientException("Failed to create store directory: " + storeDir.getAbsolutePath());
            }
            
        } catch (Exception e) {
            throw new QuickFixClientException("Failed to create required directories", e);
        }
    }

    private void addCustomFieldsToMessage(Message message) {
        for (Map.Entry<String, String> entry : customFields.entrySet()) {
            try {
                message.setString(Integer.parseInt(entry.getKey()), entry.getValue());
            } catch (NumberFormatException e) {
                logger.warn("Invalid custom field key (not numeric): {}", entry.getKey());
            }
        }
    }
}

// Example usage class
// QuickFixClientExample.java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import quickfix.Message;
import quickfix.SessionID;

public class QuickFixClientExample {
    private static final Logger logger = LoggerFactory.getLogger(QuickFixClientExample.class);

    public static void main(String[] args) {
        // Trading session example
        ConnectionDetails tradingDetails = new ConnectionDetails.Builder()
            .host("localhost")
            .port(9876)
            .senderCompId("TRADER1")
            .targetCompId("EXCHANGE")
            .username("trader_user")
            .password("trader_pass")
            .fixVersion("FIX.4.4")
            .heartbeatInterval(30)
            .resetOnLogon(true)
            .logFileDirectory("./logs/trading")
            .storeFileDirectory("./store/trading")
            .build();

        QuickFixClient tradingClient = new QuickFixClient(tradingDetails, SessionType.TRADING);
        
        // Set up listeners
        tradingClient.setMessageListener(new MessageListener() {
            @Override
            public void onMessage(SessionID sessionId, Message message) {
                logger.info("Received trading message: {}", message);
            }
        });
        
        tradingClient.setSessionEventListener(new SessionEventListener() {
            @Override
            public void onLogon(SessionID sessionId) {
                logger.info("Trading session logged on: {}", sessionId);
                
                // Send a sample order after logon
                try {
                    tradingClient.sendNewOrderSingle("EURUSD", '1', 100000, 1.1250, '2', "ORDER001");
                } catch (QuickFixClientException e) {
                    logger.error("Failed to send order", e);
                }
            }
            
            @Override
            public void onLogout(SessionID sessionId) {
                logger.info("Trading session logged out: {}", sessionId);
            }
            
            @Override
            public void onReject(SessionID sessionId, String reason) {
                logger.warn("Trading session rejected: {}", reason);
            }
            
            @Override
            public void onError(SessionID sessionId, String error) {
                logger.error("Trading session error: {}", error);
            }
        });

        // Quote session example
        ConnectionDetails quoteDetails = new ConnectionDetails.Builder()
            .host("localhost")
            .port(9877)
            .senderCompId("QUOTER1")
            .targetCompId("EXCHANGE")
            .username("quote_user")
            .password("quote_pass")
            .fixVersion("FIX.4.4")
            .heartbeatInterval(30)
            .logFileDirectory("./logs/quotes")
            .storeFileDirectory("./store/quotes")
            .build();

        QuickFixClient quoteClient = new QuickFixClient(quoteDetails, SessionType.QUOTE);
        
        quoteClient.setMessageListener(new MessageListener() {
            @Override
            public void onMessage(SessionID sessionId, Message message) {
                logger.info("Received quote message: {}", message);
            }
        });
        
        quoteClient.setSessionEventListener(new SessionEventListener() {
            @Override
            public void onLogon(SessionID sessionId) {
                logger.info("Quote session logged on: {}", sessionId);
                
                try {
                    // Request market data
                    quoteClient.sendMarketDataRequest("MDR001", '1', 1, "EURUSD", "GBPUSD");
                    // Request quote
                    quoteClient.sendQuoteRequest("QR001", "EURUSD");
                } catch (QuickFixClientException e) {
                    logger.error("Failed to send requests", e);
                }
            }
            
            @Override
            public void onLogout(SessionID sessionId) {
                logger.info("Quote session logged out: {}", sessionId);
            }
            
            @Override
            public void onReject(SessionID sessionId, String reason) {
                logger.warn("Quote session rejected: {}", reason);
            }
            
            @Override
            public void onError(SessionID sessionId, String error) {
                logger.error("Quote session error: {}", error);
            }
        });

        try {
            // Connect both sessions
            logger.info("Connecting trading client...");
            tradingClient.connect();
            
            logger.info("Connecting quote client...");
            quoteClient.connect();
            
            // Keep running for demo
            Thread.sleep(60000);
            
        } catch (Exception e) {
            logger.error("Application error", e);
        } finally {
            // Cleanup
            tradingClient.disconnect();
            quoteClient.disconnect();
        }
    }
}

// Maven dependencies (pom.xml excerpt)
/*
<dependencies>
    <dependency>
        <groupId>org.quickfixj</groupId>
        <artifactId>quickfixj-core</artifactId>
        <version>2.3.1</version>
    </dependency>
    <dependency>
        <groupId>org.quickfixj</groupId>
        <artifactId>quickfixj-messages-fix44</artifactId>
        <version>2.3.1</version>
    </dependency>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>1.7.36</version>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.2.12</version>
    </dependency>
</dependencies>
*/


```
