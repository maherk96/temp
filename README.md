```java
/**
 * Result of health check containing connection status and heartbeat information
 */
public static class HealthCheckResult {
    private final List<ConnectedClientInfo> connectedClients;
    private final List<String> disconnectedClients;
    private final long checkTimestamp;

    public HealthCheckResult(List<ConnectedClientInfo> connectedClients, 
                           List<String> disconnectedClients, 
                           long checkTimestamp) {
        this.connectedClients = new ArrayList<>(connectedClients);
        this.disconnectedClients = new ArrayList<>(disconnectedClients);
        this.checkTimestamp = checkTimestamp;
    }

    public List<ConnectedClientInfo> getConnectedClients() { return connectedClients; }
    public List<String> getDisconnectedClients() { return disconnectedClients; }
    public long getCheckTimestamp() { return checkTimestamp; }
    public int getConnectedCount() { return connectedClients.size(); }
    public int getDisconnectedCount() { return disconnectedClients.size(); }
}

/**
 * Information about a connected client including heartbeat data
 */
public static class ConnectedClientInfo {
    private final String clientName;
    private final boolean tradeSessionConnected;
    private final boolean quoteSessionConnected;
    private final Long lastTradeHeartbeat;
    private final Long lastQuoteHeartbeat;
    private final long connectionDuration;

    public ConnectedClientInfo(String clientName, 
                             boolean tradeSessionConnected, 
                             boolean quoteSessionConnected,
                             Long lastTradeHeartbeat, 
                             Long lastQuoteHeartbeat,
                             long connectionDuration) {
        this.clientName = clientName;
        this.tradeSessionConnected = tradeSessionConnected;
        this.quoteSessionConnected = quoteSessionConnected;
        this.lastTradeHeartbeat = lastTradeHeartbeat;
        this.lastQuoteHeartbeat = lastQuoteHeartbeat;
        this.connectionDuration = connectionDuration;
    }

    public String getClientName() { return clientName; }
    public boolean isTradeSessionConnected() { return tradeSessionConnected; }
    public boolean isQuoteSessionConnected() { return quoteSessionConnected; }
    public Long getLastTradeHeartbeat() { return lastTradeHeartbeat; }
    public Long getLastQuoteHeartbeat() { return lastQuoteHeartbeat; }
    public long getConnectionDuration() { return connectionDuration; }
    
    public long getTimeSinceLastTradeHeartbeat() {
        return lastTradeHeartbeat != null ? System.currentTimeMillis() - lastTradeHeartbeat : -1;
    }
    
    public long getTimeSinceLastQuoteHeartbeat() {
        return lastQuoteHeartbeat != null ? System.currentTimeMillis() - lastQuoteHeartbeat : -1;
    }
    
    public boolean isHeartbeatStale(long staleThresholdMs) {
        long currentTime = System.currentTimeMillis();
        
        if (tradeSessionConnected && lastTradeHeartbeat != null) {
            if ((currentTime - lastTradeHeartbeat) > staleThresholdMs) {
                return true;
            }
        }
        
        if (quoteSessionConnected && lastQuoteHeartbeat != null) {
            if ((currentTime - lastQuoteHeartbeat) > staleThresholdMs) {
                return true;
            }
        }
        
        return false;
    }
}

// Add these fields to FixClientPoolManager class
private final Map<SessionID, Long> lastHeartbeatTimes = new ConcurrentHashMap<>();
private final Map<String, Long> clientConnectionTimes = new ConcurrentHashMap<>();

/**
 * Perform health check and return detailed connection status
 *
 * @return HealthCheckResult containing connected clients with heartbeat info
 */
public HealthCheckResult performHealthCheck() {
    long checkTime = System.currentTimeMillis();
    List<ConnectedClientInfo> connectedClients = new ArrayList<>();
    List<String> disconnectedClients = new ArrayList<>();
    
    for (Map.Entry<String, ManagedFixClient> entry : clientPool.entrySet()) {
        String clientName = entry.getKey();
        ManagedFixClient client = entry.getValue();
        ClientStatus status = client.getStatus();
        
        if (client.isFullyConnected()) {
            // Get heartbeat times for this client's sessions
            Long lastTradeHeartbeat = getLastHeartbeatForSession(status.getTradeSessionId());
            Long lastQuoteHeartbeat = status.getQuoteSessionId() != null ? 
                getLastHeartbeatForSession(status.getQuoteSessionId()) : null;
            
            // Calculate connection duration
            Long connectionStartTime = clientConnectionTimes.get(clientName);
            long connectionDuration = connectionStartTime != null ? 
                checkTime - connectionStartTime : 0;
            
            ConnectedClientInfo clientInfo = new ConnectedClientInfo(
                clientName,
                status.isTradeSessionConnected(),
                status.isQuoteSessionConnected(),
                lastTradeHeartbeat,
                lastQuoteHeartbeat,
                connectionDuration
            );
            
            connectedClients.add(clientInfo);
            
            // Log warnings for stale heartbeats
            if (clientInfo.isHeartbeatStale(60000)) { // 60 second threshold
                logger.warn("Health check: Stale heartbeat detected for client {} - Trade: {}ms ago, Quote: {}ms ago",
                           clientName, 
                           clientInfo.getTimeSinceLastTradeHeartbeat(),
                           clientInfo.getTimeSinceLastQuoteHeartbeat());
            }
            
        } else {
            disconnectedClients.add(clientName);
            logger.warn("Health check: Client {} is not fully connected - Trade: {}, Quote: {}",
                       clientName, 
                       status.isTradeSessionConnected(),
                       status.isQuoteSessionConnected());
        }
    }
    
    logger.info("Health check completed - Connected: {}, Disconnected: {}", 
               connectedClients.size(), disconnectedClients.size());
    
    return new HealthCheckResult(connectedClients, disconnectedClients, checkTime);
}

/**
 * Get the last heartbeat time for a specific session
 */
private Long getLastHeartbeatForSession(SessionID sessionId) {
    return sessionId != null ? lastHeartbeatTimes.get(sessionId) : null;
}

/**
 * Record heartbeat for a session (call this from your SessionEventListener)
 */
public void recordHeartbeat(SessionID sessionId) {
    lastHeartbeatTimes.put(sessionId, System.currentTimeMillis());
    logger.debug("Recorded heartbeat for session: {}", sessionId);
}

/**
 * Record client connection time
 */
public void recordClientConnection(String clientName) {
    clientConnectionTimes.put(clientName, System.currentTimeMillis());
    logger.debug("Recorded connection time for client: {}", clientName);
}

/**
 * Enhanced SessionEventListener that tracks heartbeats
 */
private SessionEventListener createHeartbeatTrackingListener() {
    return new SessionEventListener() {
        @Override
        public void onLogon(SessionID sessionId) {
            String clientName = getClientNameForSession(sessionId);
            if (clientName != null) {
                recordClientConnection(clientName);
            }
            
            if (globalSessionEventListener != null) {
                globalSessionEventListener.onLogon(sessionId);
            }
        }

        @Override
        public void onLogout(SessionID sessionId) {
            // Remove heartbeat tracking when session logs out
            lastHeartbeatTimes.remove(sessionId);
            
            if (globalSessionEventListener != null) {
                globalSessionEventListener.onLogout(sessionId);
            }
        }

        @Override
        public void onReject(SessionID sessionId, String reason) {
            if (globalSessionEventListener != null) {
                globalSessionEventListener.onReject(sessionId, reason);
            }
        }
        
        // Add this method to track heartbeats
        public void onHeartbeat(SessionID sessionId) {
            recordHeartbeat(sessionId);
        }
    };
}

/**
 * Get client name from session ID
 */
private String getClientNameForSession(SessionID sessionId) {
    for (Map.Entry<String, ManagedFixClient> entry : clientPool.entrySet()) {
        ClientStatus status = entry.getValue().getStatus();
        if (sessionId.equals(status.getTradeSessionId()) || 
            sessionId.equals(status.getQuoteSessionId())) {
            return entry.getKey();
        }
    }
    return null;
}

/**
 * Get detailed health report as formatted string
 */
public String getHealthReport() {
    HealthCheckResult result = performHealthCheck();
    StringBuilder report = new StringBuilder();
    
    report.append("=== FIX Client Pool Health Report ===\n");
    report.append("Check Time: ").append(new Date(result.getCheckTimestamp())).append("\n");
    report.append("Connected Clients: ").append(result.getConnectedCount()).append("\n");
    report.append("Disconnected Clients: ").append(result.getDisconnectedCount()).append("\n\n");
    
    if (!result.getConnectedClients().isEmpty()) {
        report.append("Connected Clients Details:\n");
        for (ConnectedClientInfo client : result.getConnectedClients()) {
            report.append("  ").append(client.getClientName()).append(":\n");
            report.append("    Trade Session: ").append(client.isTradeSessionConnected() ? "UP" : "DOWN");
            if (client.getLastTradeHeartbeat() != null) {
                report.append(" (Last HB: ").append(client.getTimeSinceLastTradeHeartbeat()).append("ms ago)");
            }
            report.append("\n");
            
            // Only show quote session info if client actually has one configured
            if (hasQuoteSession(client.getClientName())) {
                report.append("    Quote Session: ").append(client.isQuoteSessionConnected() ? "UP" : "DOWN");
                if (client.getLastQuoteHeartbeat() != null) {
                    report.append(" (Last HB: ").append(client.getTimeSinceLastQuoteHeartbeat()).append("ms ago)");
                }
                report.append("\n");
            }
            
            report.append("    Connection Duration: ").append(client.getConnectionDuration() / 1000).append("s\n");
        }
    }
    
    if (!result.getDisconnectedClients().isEmpty()) {
        report.append("\nDisconnected Clients: ").append(String.join(", ", result.getDisconnectedClients())).append("\n");
    }
    
    return report.toString();
}
```
