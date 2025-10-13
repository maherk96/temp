```java
package com.yourpackage.utils;

import com.yourpackage.exceptions.NotFoundException;

import java.util.Optional;
import java.util.function.*;

/**
 * Utility class for resolving references (entities) with support for
 * caching, database fetching, and consistent NotFound handling.
 */
public final class ReferenceUtil {

    private ReferenceUtil() {
        // Utility class; prevent instantiation
    }

    /**
     * Conditionally creates an entity if it meets a given predicate.
     *
     * @param entity The entity to check.
     * @param hasContent Predicate to determine if creation should occur.
     * @param creator Function that creates and returns the ID.
     * @return The created ID, or null if not created.
     */
    public static <T> Long createIfPresent(
            T entity,
            Predicate<T> hasContent,
            Function<T, Long> creator
    ) {
        return hasContent.test(entity) ? creator.apply(entity) : null;
    }

    /**
     * Resolves a reference by ID, throwing a NotFoundException if missing.
     *
     * @param id The ID to resolve.
     * @param finder Function to find the entity by ID.
     * @param notFoundSupplier Supplier of the NotFoundException.
     * @return The resolved entity.
     */
    public static <I, E> E resolveReference(
            I id,
            Function<I, E> finder,
            Supplier<NotFoundException> notFoundSupplier
    ) {
        if (id == null) return null;
        try {
            return finder.apply(id);
        } catch (Exception e) {
            throw notFoundSupplier.get();
        }
    }

    /**
     * Resolves a reference by ID, using a cache first and then a database fetcher.
     *
     * @param idSupplier Supplies the ID to resolve.
     * @param cacheGetter Retrieves the entity from cache.
     * @param dbFetcher Retrieves the entity from the database (returns Optional).
     * @param label A label for exception messages (e.g., "Application").
     * @return The resolved entity, or null if the ID is null.
     */
    public static <T> T resolveReference(
            Supplier<Long> idSupplier,
            Function<Long, T> cacheGetter,
            Function<Long, Optional<T>> dbFetcher,
            String label
    ) {
        var id = idSupplier.get();
        if (id == null) return null;

        T cached = cacheGetter.apply(id);
        if (cached != null) return cached;

        return dbFetcher.apply(id)
                .orElseThrow(() ->
                        new NotFoundException(String.format("%s %s was not found", label, id))
                );
    }

    /**
     * Resolves a reference directly via a data fetcher (no cache, no Optional).
     *
     * @param idSupplier Supplies the ID.
     * @param dbFetcher Fetches the entity directly (throws if not found).
     * @param label Label for logging or exception messages.
     * @return The resolved entity, or null if ID is null.
     */
    public static <T> T resolveReference(
            Supplier<Long> idSupplier,
            Function<Long, T> dbFetcher,
            String label
    ) {
        var id = idSupplier.get();
        if (id == null) return null;

        try {
            return dbFetcher.apply(id);
        } catch (Exception e) {
            throw new NotFoundException(String.format("%s %s was not found", label, id));
        }
    }
}

/**
 * Maps between {@link TestLaunch} entities and {@link TestLaunchDTO} objects.
 * <p>
 * Uses MapStruct for 1:1 field mappings and custom post-processing hooks
 * to resolve references (App, Env, TestClass, TestFeature, User)
 * using caching and repository lookups.
 */
@Mapper(componentModel = "spring", nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
public abstract class TestLaunchMapper {

    @Autowired
    protected ApplicationRepository applicationRepository;

    @Autowired
    protected TestFeatureRepository testFeatureRepository;

    @Autowired
    protected UserManagementService userService;

    @Autowired
    protected EnvironmentCachedService environmentCachedService;

    @Autowired
    protected TestClassCachedService testClassCachedService;

    // ---------------------------------------------------------------
    // Core mappings
    // ---------------------------------------------------------------

    @Mapping(target = "app", ignore = true)
    @Mapping(target = "environment", ignore = true)
    @Mapping(target = "testClass", ignore = true)
    @Mapping(target = "testFeature", ignore = true)
    @Mapping(target = "user", ignore = true)
    public abstract TestLaunch toEntity(TestLaunchDTO dto);

    @Mapping(target = "app", ignore = true)
    @Mapping(target = "env", ignore = true)
    @Mapping(target = "testClass", ignore = true)
    @Mapping(target = "testFeature", ignore = true)
    @Mapping(target = "user", ignore = true)
    public abstract TestLaunchDTO toDTO(TestLaunch entity);

    // ---------------------------------------------------------------
    // After-mapping hooks (DTO → Entity)
    // ---------------------------------------------------------------

    @AfterMapping
    protected void mapRelations(TestLaunchDTO dto, @MappingTarget TestLaunch entity) {

        // --- Application (cache + DB) ---
        Application app = ReferenceUtil.resolveReference(
                dto::getApp,
                id -> ApplicationCache.getInstance().get(id),
                applicationRepository::findById,
                "Application"
        );
        entity.setApp(app);

        // --- Environment (cache only) ---
        Environment env = ReferenceUtil.resolveReference(
                dto::getEnv,
                environmentCachedService::getEnvironmentById,
                environmentCachedService::getEnvironmentById,
                "Environment"
        );
        entity.setEnvironment(env);

        // --- Test Class (cache only) ---
        TestClass testClass = ReferenceUtil.resolveReference(
                dto::getTestClass,
                testClassCachedService::getTestClassById,
                testClassCachedService::getTestClassById,
                "TestClass"
        );
        entity.setTestClass(testClass);

        // --- Test Feature (DB only) ---
        TestFeature feature = ReferenceUtil.resolveReference(
                dto.getTestFeature(),
                testFeatureRepository::findById,
                () -> new NotFoundException("TestFeature not found")
        );
        entity.setTestFeature(feature);

        // --- User (service lookup) ---
        User user = ReferenceUtil.resolveReference(
                dto::getUser,
                userService::getUserById,
                "User"
        );
        entity.setUser(user);
    }

    // ---------------------------------------------------------------
    // After-mapping hooks (Entity → DTO)
    // ---------------------------------------------------------------

    @AfterMapping
    protected void mapReverseRelations(TestLaunch entity, @MappingTarget TestLaunchDTO dto) {
        dto.setApp(entity.getApp() != null ? entity.getApp().getId() : null);
        dto.setEnv(entity.getEnvironment() != null ? entity.getEnvironment().getId() : null);
        dto.setTestClass(entity.getTestClass() != null ? entity.getTestClass().getId() : null);
        dto.setTestFeature(entity.getTestFeature() != null ? entity.getTestFeature().getId() : null);
        dto.setUser(entity.getUser() != null ? entity.getUser().getId() : null);
    }

    // ---------------------------------------------------------------
    // Partial updates
    // ---------------------------------------------------------------

    /**
     * Updates a {@link TestLaunch} entity from a {@link TestLaunchDTO},
     * ignoring {@code null} values.
     */
    public abstract void updateEntityFromDto(TestLaunchDTO dto, @MappingTarget TestLaunch entity);
}

```
