```java

public void invalidateCacheForApp(Long appId) {

    // Remove from cache
    analysisCache.asMap().keySet()
            .stream()
            .filter(key -> key.appId().equals(appId))
            .forEach(analysisCache::invalidate);

    // Remove from active key registry
    activeCacheKeys.removeIf(key -> key.appId().equals(appId));

    log.info("Invalidated all cached heatmap entries for appId={}", appId);
}
```
