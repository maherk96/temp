```java


/**
 * Service class responsible for managing tag-related operations, primarily acting as a caching
 * layer over the {@link TagService}. It uses Spring's caching annotations to improve performance
 * for repeated tag lookups and to reduce redundant database access.
 *
 * <p>This class is ideal for use in higher-level operations that frequently access or create tags,
 * ensuring that tag retrievals are cached while still supporting cache eviction upon updates.</p>
 *
 * <h2>Responsibilities:</h2>
 * <ul>
 *   <li>Delegates tag creation and retrieval to {@link TagService}</li>
 *   <li>Applies caching to repeated tag lookups</li>
 *   <li>Ensures cache consistency after tag creation or modification</li>
 * </ul>
 *
 * <h2>Caching Strategy:</h2>
 * <ul>
 *   <li>Cache name: <strong>"tagCache"</strong></li>
 *   <li>Cached operations:
 *     <ul>
 *       <li>{@link #getOrCreateTag(String, long)} – caches the returned {@link TagDTO}</li>
 *     </ul>
 *   </li>
 *   <li>Eviction operations:
 *     <ul>
 *       <li>{@link #create(TagDTO)} – clears the cache after tag creation</li>
 *     </ul>
 *   </li>
 * </ul>
 *
 * <p><strong>Thread-safety:</strong> This service is stateless and thread-safe.</p>
 *
 * @author Maher Karim
 */
@Service
public class TagManagementService {

    private final TagService tagService;

    /**
     * Constructs a new {@link TagManagementService} with the provided {@link TagService}.
     *
     * @param tagService the underlying tag service used to delegate tag operations
     */
    @Autowired
    public TagManagementService(TagService tagService) {
        this.tagService = tagService;
    }

    /**
     * Retrieves a tag by name and application ID, or creates it if it does not exist.
     * <p>The result is cached using a unique key based on the tag name and app ID combination.</p>
     *
     * @param tagName the tag name to find or create
     * @param appId   the application ID the tag belongs to
     * @return a {@link TagDTO} representing the existing or newly created tag
     */
    @Cacheable(cacheNames = "tagCache", key = "'getOrCreateTag_' + #tagName + '_' + #appId")
    public TagDTO getOrCreateTag(String tagName, long appId) {
        return tagService.getOrCreateTag(tagName, appId);
    }

    /**
     * Creates a new tag in the system and clears the tag cache to ensure
     * future lookups reflect the updated state.
     *
     * @param tagDTO the {@link TagDTO} containing the tag details to create
     * @return the ID of the newly created tag
     */
    @CacheEvict(cacheNames = "tagCache", allEntries = true)
    public long create(final TagDTO tagDTO) {
        return tagService.create(tagDTO).getId();
    }

    /**
     * Clears all cached tag entries. This can be called manually if bulk updates or
     * schema changes occur that invalidate cached tags.
     */
    @CacheEvict(cacheNames = "tagCache", allEntries = true)
    public void clearCache() {
        // Intentionally empty – triggers eviction of all cache entries
    }
}



```
