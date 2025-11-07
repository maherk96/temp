```java

@Transactional
public TagDTO getOrCreateTag(String tagName, long appId) {
    return tagRepository
        .findByNameIgnoreCaseAndAppId(tagName, appId)
        .map(tagMapper::toDTO)
        .orElseGet(() -> {
            try {
                // Try to create the tag
                TagDTO dto = TagDTO.createTag(tagName, appId);
                return create(dto);
            } catch (DataIntegrityViolationException e) {
                // Another thread likely created the same tag in parallel
                log.warn("Tag already exists for appId={} name='{}' — retrying lookup", appId, tagName);
                return tagRepository.findByNameIgnoreCaseAndAppId(tagName, appId)
                                    .map(tagMapper::toDTO)
                                    .orElseThrow(() -> new IllegalStateException(
                                        "Failed to find tag after integrity violation for appId=" + appId + " name=" + tagName
                                    ));
            }
        });
}

```
