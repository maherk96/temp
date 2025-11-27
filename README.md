```java
@Modifying
@Transactional
default void bulkInsert(Long heatmapId, Collection<Long> tagIds) {
    if (tagIds.isEmpty()) return;

    String values = tagIds.stream()
            .map(tagId -> "(" + heatmapId + ", " + tagId + ")")
            .collect(Collectors.joining(", "));

    entityManager.createNativeQuery(
            "INSERT INTO heatmap_tags (heatmap_id, tag_id) VALUES " + values
    ).executeUpdate();
}
```
