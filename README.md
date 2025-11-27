```java

@Transactional
public void createMultipleHeatmapTags(Long heatmapId, Collection<Long> tagIds) {
    if (tagIds == null || tagIds.isEmpty()) {
        return;
    }

    heatmapTagRepository.bulkInsertHeatmapTags(heatmapId, tagIds);
}

@Modifying
@Transactional
@Query("""
    INSERT INTO HeatmapTag (heatmap_id, tag_id)
    SELECT :heatmapId, t.id
    FROM Tag t
    WHERE t.id IN :tagIds
""")
void bulkInsertHeatmapTags(@Param("heatmapId") Long heatmapId,
                           @Param("tagIds") Collection<Long> tagIds);

```
