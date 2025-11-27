```java

    private record AnalysisCacheKey(
            Long appId,
            LocalDateTime startDate,
            LocalDateTime endDate,
            List<Long> sortedTagIds,
            boolean regression
    ) {}

    /** Store of cached responses for analysis queries. */
    private final Cache<AnalysisCacheKey, List<TestTagStatisticsData>> analysisCache =
            Caffeine.newBuilder()
                    .expireAfterWrite(Duration.ofMinutes(30))
                    .maximumSize(10_000)
                    .build();

    /** Track active cache keys to support selective invalidation. */
    private final Set<AnalysisCacheKey> activeCacheKeys =
            Collections.synchronizedSet(new HashSet<>());

    public List<TestTagStatisticsData> getHeatmapAnalysisData(Long appId,
                                                              LocalDateTime startDate,
                                                              LocalDateTime endDate,
                                                              Long[] tagIds,
                                                              boolean regression) {

        AnalysisCacheKey key = buildCacheKey(appId, startDate, endDate, tagIds, regression);

        return analysisCache.get(key, k -> {
            activeCacheKeys.add(k);
            return loadHeatmapAnalysisData(appId, startDate, endDate, tagIds, regression);
        });
    }

    public HeatmapDetailsDTO getHeatmapById(long id,
                                            LocalDateTime startDate,
                                            LocalDateTime endDate,
                                            boolean regression) {

        try {
            List<Long> tagIds =
                    heatmapTagManagementService.getTagIdsByHeatmapId(id);

            Heatmap heatmap =
                    heatmapManagementService.findEntityById(id);

            Long appId = heatmap.getApp().getId();

            // 🔥 Cached version
            List<TestTagStatisticsData> testTagData =
                    getHeatmapAnalysisData(appId, startDate, endDate,
                            tagIds.toArray(new Long[0]), regression);

            HeatmapResponseDTO header = buildHeaderDto(heatmap, tagIds);
            List<HeatmapAnalysisDTO> analysis =
                    buildHeatmapResponse(testTagData, startDate);

            return new HeatmapDetailsDTO(header, analysis);

        } catch (Exception e) {
            log.error("Error generating heatmap {}: {}", id, e.getMessage(), e);
            throw new HeatmapProcessingException(
                    "Failed to generate heatmap data for id " + id, e);
        }
    }

    /** Call when a NEW TEST RUN is inserted. */
    public void invalidateForNewRun(Long appId,
                                    LocalDateTime runTime,
                                    List<Long> runTagIds) {

        List<Long> sortedTags = runTagIds.stream().sorted().collect(Collectors.toList());

        for (AnalysisCacheKey key : new HashSet<>(activeCacheKeys)) {

            if (!key.appId.equals(appId)) continue;

            // Run time not inside cached range
            if (runTime.isBefore(key.startDate) || runTime.isAfter(key.endDate)) continue;

            // Tag overlap?
            boolean hasOverlap = !Collections.disjoint(key.sortedTagIds, sortedTags);
            if (!hasOverlap) continue;

            // Invalidate
            analysisCache.invalidate(key);
            activeCacheKeys.remove(key);
        }
    }

    /** Call when user updates heatmap tags. */
    public void invalidateForHeatmapTagChange(Long appId, List<Long> newTagIds) {

        List<Long> sorted = newTagIds.stream().sorted().collect(Collectors.toList());

        for (AnalysisCacheKey key : new HashSet<>(activeCacheKeys)) {
            if (key.appId.equals(appId) && !key.sortedTagIds.equals(sorted)) {
                analysisCache.invalidate(key);
                activeCacheKeys.remove(key);
            }
        }
    }

private AnalysisCacheKey buildCacheKey(Long appId,
                                           LocalDateTime startDate,
                                           LocalDateTime endDate,
                                           Long[] tagIds,
                                           boolean regression) {

        List<Long> sortedTags =
                Arrays.stream(tagIds)
                        .sorted()
                        .collect(Collectors.toList());

        return new AnalysisCacheKey(appId, startDate, endDate, sortedTags, regression);



```
