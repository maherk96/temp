```java

@Modifying
@Transactional
@Query("DELETE FROM HeatmapTag ht WHERE ht.heatmap.id = :heatmapId AND ht.tag.id IN :tagIds")
void deleteByHeatmapIdAndTagIdIn(@Param("heatmapId") Long heatmapId,
                                 @Param("tagIds") Collection<Long> tagIds);

```
