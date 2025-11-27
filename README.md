```java

@Transactional
default void bulkInsert(Long heatmapId, Collection<Long> tagIds, EntityManager em) {
    if (tagIds == null || tagIds.isEmpty()) return;

    String selectStatements = tagIds.stream()
            .map(tagId -> "SELECT " + heatmapId + ", " + tagId + " FROM dual")
            .collect(Collectors.joining(" UNION ALL "));

    String sql = "INSERT INTO heatmap_tags (heatmap_id, tag_id) " + selectStatements;

    em.createNativeQuery(sql).executeUpdate();
}
```
