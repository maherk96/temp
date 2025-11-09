```java
package com.example.service;

import com.example.domain.Tag;
import com.example.dto.TagDTO;
import com.example.exception.NotFoundException;
import com.example.mapper.TagMapper;
import com.example.repository.TagRepository;
import com.example.service.app.ApplicationManagementService;
import com.example.util.ServiceUtil;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

/**
 * Handles the physical creation of {@link Tag} entities in their own transaction scope.
 * <p>
 * This service isolates tag insertion logic using {@link Propagation#REQUIRES_NEW} to prevent
 * transactional pollution when unique constraint violations occur under concurrent access.
 * </p>
 *
 * <p>
 * Used internally by {@link TagService} to safely create tags while allowing other threads
 * to retry without impacting the current Hibernate Session.
 * </p>
 */
@Service
public class TagCreationService {

    private static final Logger log = LoggerFactory.getLogger(TagCreationService.class);

    private final TagRepository tagRepository;
    private final ApplicationManagementService appService;
    private final TagMapper tagMapper;

    public TagCreationService(TagRepository tagRepository,
                              ApplicationManagementService appService,
                              TagMapper tagMapper) {
        this.tagRepository = tagRepository;
        this.appService = appService;
        this.tagMapper = tagMapper;
    }

    /**
     * Creates a new tag in a dedicated transaction, ensuring that unique constraint violations
     * do not affect the caller's transactional state.
     *
     * @param dto the {@link TagDTO} containing tag details
     * @return the created {@link TagDTO}
     * @throws NotFoundException if the application referenced by the tag does not exist
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
    public TagDTO create(TagDTO dto) {
        Tag tag = tagMapper.toEntity(dto);

        var app = ServiceUtil.resolveReference(
                dto.getAppId(),
                appService::findByID,
                () -> new NotFoundException("Application with ID " + dto.getAppId() + " was not found")
        );

        tag.setApp(app);
        Tag saved = tagRepository.saveAndFlush(tag); // Forces flush to surface constraint errors here

        log.debug("Created new Tag '{}' for appId={}", saved.getName(), saved.getApp().getId());
        return tagMapper.toDTO(saved);
    }
}

package com.example.service;

import com.example.domain.Tag;
import com.example.dto.TagDTO;
import com.example.exception.NotFoundException;
import com.example.mapper.TagMapper;
import com.example.repository.TagRepository;
import com.example.service.app.ApplicationManagementService;
import com.example.util.ServiceUtil;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

/**
 * Provides core business logic for tag operations including retrieval, creation,
 * and concurrency-safe "get or create" semantics.
 * <p>
 * This service is the main entry point for tag operations and delegates actual creation
 * to {@link TagCreationService}, which isolates tag inserts in independent transactions.
 * </p>
 */
@Service
public class TagService {

    private static final Logger log = LoggerFactory.getLogger(TagService.class);

    private final TagRepository tagRepository;
    private final ApplicationManagementService appService;
    private final TagMapper tagMapper;
    private final TagCreationService tagCreationService;

    public TagService(TagRepository tagRepository,
                      ApplicationManagementService appService,
                      TagMapper tagMapper,
                      TagCreationService tagCreationService) {
        this.tagRepository = tagRepository;
        this.appService = appService;
        this.tagMapper = tagMapper;
        this.tagCreationService = tagCreationService;
    }

    /**
     * Returns all tags sorted by ID.
     *
     * @return list of all tags as DTOs
     */
    public List<TagDTO> findAll() {
        return tagMapper.toDTOList(tagRepository.findAllByOrderByIdAsc());
    }

    /**
     * Finds a tag by its ID and returns it as a DTO.
     *
     * @param id the tag ID
     * @return the tag DTO
     * @throws NotFoundException if tag does not exist
     */
    public TagDTO findById(Long id) {
        Tag tag = tagRepository.findById(id)
                .orElseThrow(() -> new NotFoundException("Tag not found with id: " + id));
        return tagMapper.toDTO(tag);
    }

    /**
     * Finds the entity directly (not DTO mapped).
     *
     * @param id the tag ID
     * @return the tag entity
     * @throws NotFoundException if not found
     */
    public Tag findEntityById(Long id) {
        return tagRepository.findById(id)
                .orElseThrow(() -> new NotFoundException("Tag not found with id: " + id));
    }

    /**
     * Creates a new tag using the main transactional context.
     *
     * @param dto the tag details
     * @return the created tag
     */
    @Transactional
    public TagDTO create(TagDTO dto) {
        Tag tag = tagMapper.toEntity(dto);
        var app = ServiceUtil.resolveReference(
                dto.getAppId(),
                appService::findByID,
                () -> new NotFoundException("Application with ID " + dto.getAppId() + " was not found")
        );
        tag.setApp(app);

        Tag saved = tagRepository.save(tag);
        return tagMapper.toDTO(saved);
    }

    /**
     * Retrieves an existing tag by name and app ID, or creates it if not present.
     * <p>
     * Thread-safe: multiple concurrent threads can call this safely — if multiple
     * threads attempt to create the same tag simultaneously, only one succeeds and
     * others will catch the unique constraint violation and re-query.
     * </p>
     *
     * @param tagName the tag name (case-insensitive)
     * @param appId   the application ID
     * @return an existing or newly created tag DTO
     */
    @Transactional(readOnly = true)
    public TagDTO getOrCreateTag(String tagName, long appId) {
        String normName = tagName == null ? null : tagName.trim().toLowerCase();

        return tagRepository.findByNameIgnoreCaseAndAppId(normName, appId)
                .map(tagMapper::toDTO)
                .orElseGet(() -> {
                    try {
                        TagDTO dto = TagDTO.createTag(normName, appId);
                        return tagCreationService.create(dto);
                    } catch (DataIntegrityViolationException e) {
                        log.debug("Concurrent tag creation detected for '{}'/appId={}, retrying...", normName, appId);
                        return tagRepository.findByNameIgnoreCaseAndAppId(normName, appId)
                                .map(tagMapper::toDTO)
                                .orElseThrow(() -> new IllegalStateException(
                                        String.format("Tag creation race detected but tag not found after retry (name='%s', appId=%d)",
                                                normName, appId)));
                    }
                });
    }
}

package com.example.service;

import com.example.dto.TagDTO;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

/**
 * Provides caching and high-level access for {@link TagService}.
 * <p>
 * This service wraps {@link TagService} methods with cache controls to avoid
 * redundant lookups for frequently used tag data.
 * </p>
 */
@Service
public class TagManagementService {

    private final TagService tagService;

    public TagManagementService(TagService tagService) {
        this.tagService = tagService;
    }

    /**
     * Retrieves a tag by name and application ID, or creates it if not present.
     * <p>
     * Caches results based on the combination of tag name and app ID.
     * </p>
     *
     * @param tagName the tag name
     * @param appId   the application ID
     * @return existing or newly created tag
     */
    @Cacheable(cacheNames = "tagCache", key = "'getOrCreateTag_' + #tagName + '_' + #appId")
    public TagDTO getOrCreateTag(String tagName, long appId) {
        return tagService.getOrCreateTag(tagName, appId);
    }

    /**
     * Creates a new tag and clears all cache entries to reflect updated state.
     *
     * @param tagDTO the tag to create
     * @return ID of the newly created tag
     */
    @CacheEvict(cacheNames = "tagCache", allEntries = true)
    public long create(TagDTO tagDTO) {
        return tagService.create(tagDTO).getId();
    }

    /**
     * Retrieves a tag by its ID.
     *
     * @param id tag ID
     * @return tag DTO
     */
    @Cacheable(cacheNames = "tagCache", key = "'findTagById_' + #id")
    public TagDTO findById(Long id) {
        return tagService.findById(id);
    }

    /**
     * Retrieves a tag entity by ID (no DTO mapping).
     *
     * @param id tag ID
     * @return tag entity
     */
    @Cacheable(cacheNames = "tagCache", key = "'findTagEntityById_' + #id")
    public Object findEntityById(Long id) {
        return tagService.findEntityById(id);
    }
}

package com.example.domain;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;
import org.hibernate.annotations.CreationTimestamp;

import java.time.Instant;

/**
 * Represents a unique tag associated with an application.
 * <p>
 * Each {@link Tag} belongs to one {@link Application} and is uniquely
 * identified by the combination of {@code app_id} and {@code name}.
 * </p>
 *
 * <p>
 * Tags are stored in lowercase to ensure case-insensitive uniqueness.
 * Normalization occurs automatically before persisting or updating.
 * </p>
 */
@Entity
@Table(
    name = "TAG",
    uniqueConstraints = @UniqueConstraint(columnNames = {"APP_ID", "NAME"})
)
@Getter
@Setter
public class Tag {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    /**
     * Tag name, stored in lowercase for uniqueness.
     */
    @Column(length = 50, nullable = false)
    private String name;

    /**
     * The application this tag belongs to.
     */
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "APP_ID", nullable = false)
    private Application app;

    /**
     * Timestamp of when the tag was created.
     */
    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private Instant created;

    /**
     * Normalizes the tag name before persisting or updating to enforce
     * lowercase consistency in the database.
     */
    @PrePersist
    @PreUpdate
    private void normalizeName() {
        if (this.name != null) {
            this.name = this.name.trim().toLowerCase();
        }
    }

    /**
     * Two tags are equal if they belong to the same app and have the same name.
     * This is consistent with the unique constraint at the DB level.
     */
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Tag)) return false;
        Tag other = (Tag) o;
        return app != null && app.getId() != null
                && other.app != null && other.app.getId() != null
                && name != null && name.equalsIgnoreCase(other.name)
                && app.getId().equals(other.app.getId());
    }

    @Override
    public int hashCode() {
        return (app == null || app.getId() == null || name == null)
                ? super.hashCode()
                : (app.getId() + ":" + name.toLowerCase()).hashCode();
    }

    @Override
    public String toString() {
        return "Tag{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", appId=" + (app != null ? app.getId() : null) +
                ", created=" + created +
                '}';
    }
}




```
