```java
    @Transactional
    default void bulkInsert(
            Long heatmapId,
            Collection<Long> tagIds,
            EntityManager em
    ) {
        if (tagIds == null || tagIds.isEmpty()) return;

        String selectBlocks = tagIds.stream()
                .map(tagId -> "SELECT heatmap_tags_seq.nextval, "
                        + heatmapId + ", "
                        + tagId + ", "
                        + "SYSDATE FROM dual")
                .collect(Collectors.joining(" UNION ALL "));

        String sql = "INSERT INTO heatmap_tags (id, heatmap_id, tag_id, created) "
                   + selectBlocks;

        em.createNativeQuery(sql).executeUpdate();
    }

    @PersistenceContext
    private EntityManager entityManager;

    @Transactional
    public void createMultipleHeatmapTags(Long heatmapId, Collection<Long> tagIds) {
        heatmapTagRepository.bulkInsert(heatmapId, tagIds, entityManager);
    }


```
