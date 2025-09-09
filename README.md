```java
/**
 * Stop a specific client by name
 *
 * @param clientStreamName Name of the client to stop
 * @param timeout Maximum time to wait for graceful shutdown
 * @param unit Time unit for timeout
 * @return true if client stopped gracefully, false if timeout occurred or client not found
 * @throws FixClientPoolException if pool is not started
 */
public boolean stopClient(String clientStreamName, long timeout, TimeUnit unit) throws FixClientPoolException {
    ensureStarted();
    
    ManagedFixClient client = clientPool.remove(clientStreamName);
    if (client == null) {
        logger.warn("Cannot stop client - not found: {}", clientStreamName);
        return false;
    }
    
    logger.info("Stopping client: {}", clientStreamName);
    
    CompletableFuture<Void> stopFuture = CompletableFuture.runAsync(() -> {
        try {
            client.stop();
            logger.info("Client {} stopped successfully", clientStreamName);
        } catch (Exception e) {
            logger.error("Error stopping client {}", clientStreamName, e);
            throw new RuntimeException(e);
        }
    });
    
    try {
        stopFuture.get(timeout, unit);
        
        // Reset message counter for this client
        clientMessageCounts.get(clientStreamName).set(0);
        
        logger.info("Client {} stopped gracefully", clientStreamName);
        return true;
        
    } catch (InterruptedException | ExecutionException | TimeoutException e) {
        logger.error("Failed to stop client {} within timeout", clientStreamName, e);
        
        // Force stop the client
        try {
            client.stop();
        } catch (Exception ex) {
            logger.error("Error during force stop of client {}", clientStreamName, ex);
        }
        
        return false;
    }
}

/**
 * Stop a specific client with default timeout (30 seconds)
 */
public boolean stopClient(String clientStreamName) throws FixClientPoolException {
    return stopClient(clientStreamName, 30, TimeUnit.SECONDS);
}

/**
 * Restart a specific client
 *
 * @param clientStreamName Name of the client to restart
 * @param stopTimeout Maximum time to wait for stop
 * @param startTimeout Maximum time to wait for start and connection
 * @param unit Time unit for timeouts
 * @return true if restart was successful, false otherwise
 * @throws FixClientPoolException if pool is not started or client configuration not found
 */
public boolean restartClient(String clientStreamName, long stopTimeout, long startTimeout, TimeUnit unit) 
        throws FixClientPoolException {
    ensureStarted();
    
    // Validate that this client is in our configured set
    if (!clientStreamNames.contains(clientStreamName)) {
        throw new FixClientPoolException("Client not in configured set: " + clientStreamName);
    }
    
    logger.info("Restarting client: {}", clientStreamName);
    
    try {
        // Step 1: Stop existing client if it exists
        ManagedFixClient existingClient = clientPool.get(clientStreamName);
        if (existingClient != null) {
            logger.info("Stopping existing client: {}", clientStreamName);
            if (!stopClient(clientStreamName, stopTimeout, unit)) {
                logger.error("Failed to stop existing client {}, aborting restart", clientStreamName);
                return false;
            }
        }
        
        // Step 2: Create new client instance
        logger.info("Creating new client instance: {}", clientStreamName);
        ManagedFixClient newClient = createManagedClient(clientStreamName);
        
        // Step 3: Start the new client
        logger.info("Starting new client: {}", clientStreamName);
        newClient.start();
        
        // Step 4: Add to pool
        clientPool.put(clientStreamName, newClient);
        
        // Step 5: Wait for connection
        logger.info("Waiting for client {} to connect...", clientStreamName);
        CompletableFuture<Boolean> connectionFuture = CompletableFuture.supplyAsync(() -> {
            try {
                return newClient.waitForConnection(startTimeout, unit);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException("Interrupted waiting for connection", e);
            }
        });
        
        boolean connected = connectionFuture.get(startTimeout + 5, unit); // Small buffer
        
        if (connected) {
            logger.info("Client {} restarted and connected successfully", clientStreamName);
            return true;
        } else {
            logger.error("Client {} failed to connect after restart", clientStreamName);
            // Clean up the failed client
            stopClient(clientStreamName, 10, TimeUnit.SECONDS);
            return false;
        }
        
    } catch (Exception e) {
        logger.error("Error during restart of client {}", clientStreamName, e);
        
        // Cleanup: try to remove any partially created client
        try {
            stopClient(clientStreamName, 5, TimeUnit.SECONDS);
        } catch (Exception cleanup) {
            logger.error("Error during cleanup after failed restart", cleanup);
        }
        
        throw new FixClientPoolException("Failed to restart client: " + clientStreamName, e);
    }
}

/**
 * Restart a specific client with default timeouts (30 seconds each for stop and start)
 */
public boolean restartClient(String clientStreamName) throws FixClientPoolException {
    return restartClient(clientStreamName, 30, 30, TimeUnit.SECONDS);
}

/**
 * Start a specific client that was previously stopped
 *
 * @param clientStreamName Name of the client to start
 * @param timeout Maximum time to wait for start and connection
 * @param unit Time unit for timeout
 * @return true if client started successfully
 * @throws FixClientPoolException if pool is not started, client already running, or client not configured
 */
public boolean startClient(String clientStreamName, long timeout, TimeUnit unit) throws FixClientPoolException {
    ensureStarted();
    
    // Check if client is already running
    if (clientPool.containsKey(clientStreamName)) {
        logger.warn("Client {} is already running", clientStreamName);
        return false;
    }
    
    // Validate that this client is in our configured set
    if (!clientStreamNames.contains(clientStreamName)) {
        throw new FixClientPoolException("Client not in configured set: " + clientStreamName);
    }
    
    logger.info("Starting client: {}", clientStreamName);
    
    try {
        // Create and start new client
        ManagedFixClient client = createManagedClient(clientStreamName);
        client.start();
        clientPool.put(clientStreamName, client);
        
        // Wait for connection
        CompletableFuture<Boolean> connectionFuture = CompletableFuture.supplyAsync(() -> {
            try {
                return client.waitForConnection(timeout, unit);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException("Interrupted waiting for connection", e);
            }
        });
        
        boolean connected = connectionFuture.get(timeout + 5, unit);
        
        if (connected) {
            logger.info("Client {} started and connected successfully", clientStreamName);
            return true;
        } else {
            logger.error("Client {} failed to connect after start", clientStreamName);
            stopClient(clientStreamName, 10, TimeUnit.SECONDS);
            return false;
        }
        
    } catch (Exception e) {
        logger.error("Error starting client {}", clientStreamName, e);
        try {
            stopClient(clientStreamName, 5, TimeUnit.SECONDS);
        } catch (Exception cleanup) {
            logger.error("Error during cleanup after failed start", cleanup);
        }
        throw new FixClientPoolException("Failed to start client: " + clientStreamName, e);
    }
}

/**
 * Start a specific client with default timeout (30 seconds)
 */
public boolean startClient(String clientStreamName) throws FixClientPoolException {
    return startClient(clientStreamName, 30, TimeUnit.SECONDS);
}

/**
 * Get the status of a specific client
 *
 * @param clientStreamName Name of the client
 * @return ClientStatus if client exists and is running, null otherwise
 */
public ClientStatus getClientStatus(String clientStreamName) {
    ManagedFixClient client = clientPool.get(clientStreamName);
    return client != null ? client.getStatus() : null;
}

/**
 * Check if a specific client is currently running in the pool
 */
public boolean isClientRunning(String clientStreamName) {
    return clientPool.containsKey(clientStreamName);
}

/**
 * Get list of all currently running clients
 */
public Set<String> getRunningClients() {
    return new HashSet<>(clientPool.keySet());
}

/**
 * Get list of all configured clients (running + stopped)
 */
public Set<String> getConfiguredClients() {
    return new HashSet<>(clientStreamNames);
}

/**
 * Get list of stopped clients (configured but not currently running)
 */
public Set<String> getStoppedClients() {
    Set<String> stopped = new HashSet<>(clientStreamNames);
    stopped.removeAll(clientPool.keySet());
    return stopped;
}

```
