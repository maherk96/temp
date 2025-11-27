```java

@Modifying
@Transactional
@Query(
    value = """
        INSERT INTO heatmap_tags (heatmap_id, tag_id)
        VALUES (:heatmapId, :tagId)
        """,
    nativeQuery = true
)
void insertSingle(@Param("heatmapId") Long heatmapId,
                  @Param("tagId") Long tagId);

```
