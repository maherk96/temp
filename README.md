```java
@Service
public class TagService {

    @Autowired
    private TagRepository tagRepository;

    @Autowired
    private TagMapper tagMapper;

    @PersistenceContext
    private EntityManager entityManager;

    @Transactional
    public TagDTO getOrCreateTag(String tagName, long appId) {
        // First try to find
        return tagRepository.findByNameIgnoreCaseAndAppId(tagName, appId)
                .map(tagMapper::toDTO)
                .orElseGet(() -> tryCreateTag(tagName, appId));
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public TagDTO tryCreateTag(String tagName, long appId) {
        try {
            TagDTO dto = TagDTO.createTag(tagName, appId);
            return create(dto);
        } catch (DataIntegrityViolationException e) {
            // Another thread probably inserted the same tag concurrently
            log.warn("Tag already exists for appId={} name='{}' — retrying lookup", appId, tagName);

            // Clear this session to remove the failed entity
            entityManager.clear();

            // Lookup in a new clean transaction
            return findExistingTag(tagName, appId);
        }
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW, readOnly = true)
    public TagDTO findExistingTag(String tagName, long appId) {
        return tagRepository.findByNameIgnoreCaseAndAppId(tagName, appId)
                .map(tagMapper::toDTO)
                .orElseThrow(() ->
                        new IllegalStateException("Tag not found after constraint violation: " + tagName));
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public TagDTO create(TagDTO dto) {
        Tag entity = tagMapper.toEntity(dto);
        Tag saved = tagRepository.save(entity);
        return tagMapper.toDTO(saved);
    }
}
```
