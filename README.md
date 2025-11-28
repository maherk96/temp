```java
@Slf4j
@Service
public class HeatmapCacheService {

    private final Cache<AnalysisCacheKey, List<TestTagStatisticsData>> analysisCache =
            Caffeine.newBuilder()
                    .expireAfterWrite(Duration.ofDays(1))
                    .maximumSize(10_000)
                    .build();

    private final Cache<CellDataCacheKey, List<HeatmapCellData>> cellDataCache =
            Caffeine.newBuilder()
                    .expireAfterWrite(Duration.ofDays(1))
                    .maximumSize(5_000)
                    .build();

    // ---------- Generic invalidation helper ----------
    private <K, V> void invalidate(Cache<K, V> cache, Predicate<K> filter) {
        cache.asMap().keySet().removeIf(filter);
    }

    // ---------- Public API ----------

    public List<TestTagStatisticsData> getAnalysisData(
            Long appId,
            LocalDateTime start,
            LocalDateTime end,
            Long[] tagIds,
            boolean regression,
            Supplier<List<TestTagStatisticsData>> dataLoader) {

        AnalysisCacheKey key = buildAnalysisCacheKey(appId, start, end, tagIds, regression);

        return analysisCache.get(key, k -> {
            long t = System.currentTimeMillis();
            var data = dataLoader.get();
            log.debug("Loaded analysis data for key={} in {}ms", k, System.currentTimeMillis() - t);
            return data;
        });
    }

    public List<HeatmapCellData> getCellData(
            LocalDateTime start,
            LocalDateTime end,
            String className,
            String tagName,
            String appName,
            Supplier<List<HeatmapCellData>> dataLoader) {

        CellDataCacheKey key = buildCellDataCacheKey(start, end, className, tagName, appName);

        return cellDataCache.get(key, k -> {
            long t = System.currentTimeMillis();
            var data = dataLoader.get();
            log.debug("Loaded cell data for key={} in {}ms", k, System.currentTimeMillis() - t);
            return data;
        });
    }

    public void invalidateCacheForApp(Long appId) {
        invalidate(analysisCache, key -> key.appId().equals(appId));
        log.info("Invalidated analysis cache for appId={}", appId);
    }

    public void invalidateCellDataCacheForApp(String appName) {
        invalidate(cellDataCache, key -> Objects.equals(key.appName(), appName));
        log.info("Invalidated cell data cache for appName={}", appName);
    }

    public void invalidateAllCachesForApp(Long appId, String appName) {
        invalidateCacheForApp(appId);
        invalidateCellDataCacheForApp(appName);
        log.info("Invalidated all caches for appId={}, appName={}", appId, appName);
    }

    public void invalidateCellDataCacheForClassAndTag(String className, String tagName, String appName) {
        invalidate(cellDataCache, key ->
                (className == null || Objects.equals(key.className(), className)) &&
                (tagName == null || Objects.equals(key.tagName(), tagName)) &&
                (appName == null || Objects.equals(key.appName(), appName))
        );

        log.info("Invalidated cell data for className={}, tagName={}, appName={}",
                className, tagName, appName);
    }

    public void clearAllCaches() {
        analysisCache.invalidateAll();
        cellDataCache.invalidateAll();
        log.info("Cleared all heatmap caches");
    }

    // ---------- Key builders ----------

    private AnalysisCacheKey buildAnalysisCacheKey(
            Long appId,
            LocalDateTime start,
            LocalDateTime end,
            Long[] tagIds,
            boolean regression) {

        List<Long> sortedTags = tagIds == null
                ? List.of()
                : Arrays.stream(tagIds).sorted().toList();

        return new AnalysisCacheKey(appId, start, end, sortedTags, regression);
    }

    private CellDataCacheKey buildCellDataCacheKey(
            LocalDateTime start,
            LocalDateTime end,
            String className,
            String tagName,
            String appName) {

        return new CellDataCacheKey(start, end, className, tagName, appName);
    }
}


```
