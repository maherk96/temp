```java

@Transactional
public TagDTO getOrCreateTag(String tagName, long appId) {
    try {
        return tagRepository
            .findByNameIgnoreCaseAndAppId(tagName, appId)
            .map(tagMapper::toDTO)
            .orElseGet(() -> {
                TagDTO dto = TagDTO.createTag(tagName, appId);
                return create(dto);
            });
    } catch (DataIntegrityViolationException e) {
        log.warn("Tag already exists for appId={} name='{}' — retrying lookup", appId, tagName);

        // The current transaction is marked for rollback → start a new one
        return retryLookupAfterConstraintViolation(tagName, appId);
    }
}

/**
 * Performs the retry in a new transaction since the previous one failed.
 */
@Transactional(propagation = Propagation.REQUIRES_NEW, readOnly = true)
public TagDTO retryLookupAfterConstraintViolation(String tagName, long appId) {
    return tagRepository.findByNameIgnoreCaseAndAppId(tagName, appId)
                        .map(tagMapper::toDTO)
                        .orElseThrow(() -> new IllegalStateException(
                            "Failed to find tag after integrity violation for appId=" + appId + " name=" + tagName
                        ));
}
```
