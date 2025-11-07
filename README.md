```java
@Transactional
public TagDTO getOrCreateTag(String tagName, long appId) {
    // First attempt - normal flow
    Optional<Tag> existing = tagRepository.findByNameIgnoreCaseAndAppId(tagName, appId);
    if (existing.isPresent()) {
        return tagMapper.toDTO(existing.get());
    }
    
    // Try to create
    try {
        TagDTO dto = TagDTO.createTag(tagName, appId);
        return create(dto);
    } catch (DataIntegrityViolationException e) {
        // Another thread won the race - fetch the tag they created
        // This needs to be done in a new query to see committed data
        Tag tag = tagRepository.findByNameIgnoreCaseAndAppId(tagName, appId)
            .orElseThrow(() -> new IllegalStateException(
                "Tag should exist after constraint violation but was not found"));
        return tagMapper.toDTO(tag);
    }
}
```
