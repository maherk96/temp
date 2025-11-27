```java

@Transactional
public void createMultipleHeatmapTags(Long heatmapId, Collection<Long> tagIds) {
    if (tagIds == null || tagIds.isEmpty()) return;

    tagIds.forEach(tagId ->
        heatmapTagRepository.insertSingle(heatmapId, tagId)
    );
}
```
