```java
public Optional<DefaultNewOrderSingle> getOrder(String sessionId, String orderRef) {
    var key = new OrderKey(sessionId, orderRef);
    log.info("Looking up order with key: {}", key);
    
    var manifest = orderManifestCache.get(key);
    if (manifest != null) {
        var order = manifest.getDefaultNewOrderSingle();
        log.info("Found order - lookup key: {}, order clOrdID: {}", orderRef, order.clOrdID());
        log.info("Cache contents: {}", orderManifestCache.keySet());
    } else {
        log.warn("No order found for key: {}", key);
        log.info("Cache contents: {}", orderManifestCache.keySet());
    }
    
    return Optional.ofNullable(manifest)
        .map(OrderManifest::getDefaultNewOrderSingle);
}
```
