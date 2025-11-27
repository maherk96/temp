```java
public HeatmapResponseDTO update(final Long id, final HeatmapCreate heatmapUpdate) {

    // Load entity
    final Heatmap heatmap = heatmapRepository.findById(id)
            .orElseThrow(NotFoundException::new);

    // Update main fields
    heatmap.setName(heatmapUpdate.name());
    heatmap.setDescription(heatmapUpdate.description());
    heatmap.setApp(applicationManagementService.findByID(heatmapUpdate.applicationId()));

    // ---------- TAG SYNC (Optimised) ----------

    // Convert both lists to sets for O(1) lookups
    Set<Long> existingTagIds = new HashSet<>(
            heatmapTagManagementService.getTagIdsByHeatmapId(id)
    );

    Set<Long> newTagIds = new HashSet<>(heatmapUpdate.tagIds());

    // Determine removals: existing - new
    Set<Long> tagsToRemove = new HashSet<>(existingTagIds);
    tagsToRemove.removeAll(newTagIds);

    if (!tagsToRemove.isEmpty()) {
        heatmapTagManagementService.removeHeatmapTags(id, tagsToRemove);
    }

    // Determine additions: new - existing
    Set<Long> tagsToAdd = new HashSet<>(newTagIds);
    tagsToAdd.removeAll(existingTagIds);

    if (!tagsToAdd.isEmpty()) {
        heatmapTagManagementService.createMultipleHeatmapTags(id, tagsToAdd);
    }

    // ---------- SAVE MAIN ENTITY ----------
    Heatmap updatedHeatmap = heatmapRepository.save(heatmap);

    // User
    var user = updatedHeatmap.getUser();
    String soeid = (user != null ? user.getName() : "Unknown User");

    // No second DB hit needed — we already know the final tags
    List<Long> finalTagIds = new ArrayList<>(newTagIds);

    // Callback
    onTagUpdate(heatmapUpdate.applicationId());

    // ---------- RETURN RESPONSE ----------
    return new HeatmapResponseDTO(
            updatedHeatmap.getId(),
            soeid,
            updatedHeatmap.getName(),
            updatedHeatmap.getDescription(),
            updatedHeatmap.getDeleted(),
            updatedHeatmap.getApp().getId(),
            finalTagIds
    );
}
```
