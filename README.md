```java
private final ConcurrentHashMap<String, Object> locks = new ConcurrentHashMap<>();

@Transactional
public TagDTO getOrCreateTag(String tagName, long appId) {
    String lockKey = tagName.toLowerCase() + "_" + appId;
    Object lock = locks.computeIfAbsent(lockKey, k -> new Object());
    
    synchronized (lock) {
        try {
            Optional<Tag> existing = tagRepository.findByNameIgnoreCaseAndAppId(tagName, appId);
            if (existing.isPresent()) {
                return tagMapper.toDTO(existing.get());
            }
            
            TagDTO dto = TagDTO.createTag(tagName, appId);
            return create(dto);
        } finally {
            locks.remove(lockKey);
        }
    }
}
```
