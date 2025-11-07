```java
@Transactional
public TagDTO getOrCreateTag(String tagName, long appId) {
    try {
        return tagRepository
            .findByNameIgnoreCaseAndAppId(tagName, appId)
            .map(tagMapper::toDTO)
            .orElseGet(() -> createNewTagSafe(tagName, appId));
    } catch (DataIntegrityViolationException e) {
        log.warn("Tag already exists for appId={} name='{}' — retrying lookup", appId, tagName);
        return retryLookupAfterConstraintViolation(tagName, appId);
    }
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public TagDTO createNewTagSafe(String tagName, long appId) {
    TagDTO dto = TagDTO.createTag(tagName, appId);
    return create(dto); // this will commit or fail independently
}

@Transactional(propagation = Propagation.REQUIRES_NEW, readOnly = true)
public TagDTO retryLookupAfterConstraintViolation(String tagName, long appId) {
    return tagRepository.findByNameIgnoreCaseAndAppId(tagName, appId)
                        .map(tagMapper::toDTO)
                        .orElseThrow(() ->
                            new IllegalStateException("Tag not found after integrity violation")
                        );
}
```
