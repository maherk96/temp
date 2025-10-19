```java
public void addOrder(String sessionId, String orderRef, DefaultNewOrderSingle defaultNewOrderSingle) {
    log.info("Adding order - orderRef param: {}, order.clOrdID(): {}", 
        orderRef, defaultNewOrderSingle.clOrdID());
    
    var orderKey = new OrderKey(sessionId, orderRef);
    orderManifestCache.computeIfAbsent(
        orderKey, k -> new OrderManifest(defaultNewOrderSingle, null));
    
    // IMMEDIATELY after adding, check what's stored
    var stored = orderManifestCache.get(orderKey).getDefaultNewOrderSingle();
    log.info("Stored order - key orderRef: {}, stored order.clOrdID(): {}", 
        orderRef, stored.clOrdID());
        
    orderRefToSessionIdCache.put(orderRef, sessionId);
}
```
