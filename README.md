```java
 @Transactional
    public TagDTO getOrCreateTag(String tagName, long appId) {
        return tagRepository.findByNameIgnoreCaseAndAppId(tagName, appId)
                .map(tagMapper::toDTO)
                .orElseGet(() -> {
                    try {
                        // Attempt to create a new tag
                        TagDTO tagDTO = TagDTO.createTag(tagName, appId);
                        return create(tagDTO);
                    } catch (DataIntegrityViolationException e) {
                        // Another thread/process inserted it concurrently — retrieve it
                        return tagRepository.findByNameIgnoreCaseAndAppId(tagName, appId)
                                .map(tagMapper::toDTO)
                                .orElseThrow(() -> new IllegalStateException(
                                        "Tag creation failed but tag not found afterward: " + tagName
                                ));
                    }
                });
    }
```
