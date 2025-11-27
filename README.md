```java

    @Transactional
    public void createMultipleHeatmapTagsBulk(Long heatmapId, Collection<Long> tagIds,
                                              javax.persistence.EntityManager em) {
        if (tagIds == null || tagIds.isEmpty()) return;

        heatmapTagRepository.bulkInsert(heatmapId, tagIds, em);
    }

    @Transactional
    default void bulkInsert(Long heatmapId, Collection<Long> tagIds, javax.persistence.EntityManager em) {
        if (tagIds == null || tagIds.isEmpty()) return;

        String values = tagIds.stream()
                .map(tagId -> "(" + heatmapId + ", " + tagId + ")")
                .reduce((a, b) -> a + ", " + b)
                .orElse("");

        String sql = "INSERT INTO heatmap_tags (heatmap_id, tag_id) VALUES " + values;

        em.createNativeQuery(sql).executeUpdate();
    }
```
