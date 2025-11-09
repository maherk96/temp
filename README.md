```java
@Service
public class TagService {

    private final TagRepository tagRepository;
    private final AppService appService;
    private final TagMapper tagMapper;

    public TagService(TagRepository tagRepository, AppService appService, TagMapper tagMapper) {
        this.tagRepository = tagRepository;
        this.appService = appService;
        this.tagMapper = tagMapper;
    }

    // Creates in its own transaction so a constraint failure doesn't poison the caller’s Tx
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public TagDTO create(TagDTO dto) {
        Tag tag = tagMapper.toEntity(dto);

        final var app = ServiceUtil.resolveReference(
            dto.getAppId(),
            appService::findByID,
            () -> new NotFoundException("Application with ID " + dto.getAppId() + " was not found")
        );

        tag.setApp(app);
        Tag saved = tagRepository.saveAndFlush(tag); // flush to surface constraint violations here
        return tagMapper.toDTO(saved);
    }

    // Outer method: not REQUIRED new. It remains clean if inner Tx fails.
    @Transactional(readOnly = true)
    public TagDTO getOrCreateTag(String tagName, long appId) {
        // Normalize exactly how your unique index works
        String normName = tagName == null ? null : tagName.trim();

        // 1) Try to find
        Optional<Tag> existing = tagRepository.findByNameIgnoreCaseAndAppId(normName, appId);
        if (existing.isPresent()) {
            return tagMapper.toDTO(existing.get());
        }

        // 2) Try to create in a new Tx
        try {
            TagDTO dto = TagDTO.createTag(normName, appId);
            return create(dto);
        } catch (DataIntegrityViolationException e) {
            // 3) Someone else created it concurrently; read it now (this Tx is still valid)
            return tagRepository.findByNameIgnoreCaseAndAppId(normName, appId)
                .map(tagMapper::toDTO)
                .orElseThrow(() -> new IllegalStateException(
                    "Constraint hit but tag not found after retry: name=" + normName + ", appId=" + appId
                ));
        }
    }
}

```
