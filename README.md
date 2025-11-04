```java
package com.citi.fx.qa.qap.db.services;

import com.citi.fx.qa.qap.db.dto.TagDTO;
import com.citi.fx.qa.qap.db.entities.Tag;
import com.citi.fx.qa.qap.db.entities.Application;
import com.citi.fx.qa.qap.db.repos.TagRepository;
import com.citi.fx.qa.qap.db.mappers.TagMapper;
import com.citi.fx.qa.qap.db.services.ApplicationManagementService;
import com.citi.fx.qa.qap.db.exceptions.NotFoundException;
import org.springframework.data.domain.Sort;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

/**
 * Service class for managing {@link Tag} entities.
 * <p>
 * This service provides CRUD and utility operations for the {@code TAG} table.
 * It acts as a middle layer between the persistence repository and the controller or test reporting layer.
 * </p>
 *
 * <h2>Responsibilities:</h2>
 * <ul>
 *   <li>Fetching tags from the database</li>
 *   <li>Creating and mapping {@link TagDTO} objects</li>
 *   <li>Handling application linkage between tags and applications</li>
 *   <li>Creating tags on demand when not already existing</li>
 * </ul>
 *
 * <h2>Mapping:</h2>
 * <p>All entity-to-DTO and DTO-to-entity mappings are delegated to {@link TagMapper}.</p>
 *
 * @author Maher Karim
 */
@Service
public class TagService {

    private final TagRepository tagRepository;
    private final ApplicationManagementService appService;
    private final TagMapper tagMapper;

    public TagService(TagRepository tagRepository,
                      ApplicationManagementService appService,
                      TagMapper tagMapper) {
        this.tagRepository = tagRepository;
        this.appService = appService;
        this.tagMapper = tagMapper;
    }

    /**
     * Retrieves all tags in ascending order by ID.
     *
     * @return a list of all tags as {@link TagDTO} objects
     */
    public List<TagDTO> findAll() {
        return tagMapper.toDTOList(tagRepository.findAll(Sort.by("id")));
    }

    /**
     * Finds a tag by its database ID.
     *
     * @param id the unique tag ID
     * @return a {@link TagDTO} representing the tag
     * @throws NotFoundException if no tag with the given ID exists
     */
    public TagDTO findById(Long id) {
        Tag tag = tagRepository.findById(id)
                .orElseThrow(() -> new NotFoundException("Tag not found with id: " + id));
        return tagMapper.toDTO(tag);
    }

    /**
     * Creates and persists a new tag based on the provided DTO.
     * <p>
     * If the DTO contains an {@code appId}, the corresponding {@link Application}
     * entity is looked up and linked to the tag.
     * </p>
     *
     * @param dto the tag data to persist
     * @return a DTO representation of the newly created tag
     * @throws NotFoundException if the specified application ID does not exist
     */
    @Transactional
    public TagDTO create(TagDTO dto) {
        Tag tag = tagMapper.toEntity(dto);

        if (dto.getAppId() != null) {
            Application app = appService.findById(dto.getAppId())
                    .orElseThrow(() -> new NotFoundException("Application with ID " + dto.getAppId() + " not found"));
            tag.setApp(app);
        }

        Tag saved = tagRepository.save(tag);
        return tagMapper.toDTO(saved);
    }

    /**
     * Retrieves a tag by name (case-insensitive) or creates a new one if it does not exist.
     * <p>
     * This operation is transactional to ensure that no duplicate tags are created
     * in concurrent execution scenarios.
     * </p>
     *
     * @param tagName the tag name to find or create
     * @param appId   the application ID this tag belongs to
     * @return an existing or newly created {@link TagDTO}
     */
    @Transactional
    public TagDTO getOrCreateTag(String tagName, long appId) {
        return tagRepository.findByNameIgnoreCase(tagName)
                .map(tagMapper::toDTO)
                .orElseGet(() -> {
                    TagDTO dto = TagDTO.createTag(tagName, appId);
                    return create(dto);
                });
    }
}


package com.citi.fx.qa.qap.db.mappers;

import com.citi.fx.qa.qap.db.entities.Application;
import com.citi.fx.qa.qap.db.entities.Tag;
import com.citi.fx.qa.qap.db.dto.TagDTO;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import java.util.List;

/**
 * Mapper interface for converting between {@link Tag} entities and {@link TagDTO} data transfer objects.
 * <p>
 * This mapper uses MapStruct to automatically generate type-safe implementations at compile time.
 * It simplifies conversion between entity and DTO layers, avoiding manual boilerplate mapping code.
 * </p>
 *
 * <p><strong>Mapping Rules:</strong></p>
 * <ul>
 *   <li>Entity → DTO: Maps {@code Tag.app.id} to {@code TagDTO.appId}</li>
 *   <li>DTO → Entity: Maps {@code TagDTO.appId} to a new {@code Application} reference with the same ID</li>
 *   <li>All other matching fields (e.g., {@code id}, {@code name}) are mapped automatically.</li>
 * </ul>
 *
 * <p><strong>Component Model:</strong> {@code spring}</p>
 * This allows Spring to auto-discover and inject the mapper as a bean.
 *
 * @author Maher Karim
 */
@Mapper(componentModel = "spring")
public interface TagMapper {

    /**
     * Converts a {@link Tag} entity into a {@link TagDTO}.
     *
     * @param tag the entity to convert (must not be {@code null})
     * @return a DTO representation of the entity
     */
    @Mapping(target = "appId", source = "app.id")
    TagDTO toDTO(Tag tag);

    /**
     * Converts a {@link TagDTO} into a {@link Tag} entity.
     * <p>
     * The {@code app} field is reconstructed as a lightweight {@link Application} reference
     * using only the ID value, to maintain referential integrity without fetching the full object.
     * </p>
     *
     * @param dto the DTO to convert (must not be {@code null})
     * @return a new {@link Tag} entity instance
     */
    @Mapping(target = "app", source = "appId")
    Tag toEntity(TagDTO dto);

    /**
     * Converts a list of {@link Tag} entities into a list of {@link TagDTO} objects.
     *
     * @param tags list of tag entities to convert
     * @return list of tag DTOs
     */
    List<TagDTO> toDTOList(List<Tag> tags);

    /**
     * Helper method for converting an {@link Long appId} into an {@link Application} reference.
     * <p>
     * This is used automatically by MapStruct during {@link #toEntity(TagDTO)} mapping.
     * </p>
     *
     * @param appId the application ID to wrap
     * @return an {@link Application} entity with only its ID populated, or {@code null} if {@code appId} is {@code null}
     */
    default Application map(Long appId) {
        if (appId == null) return null;
        Application app = new Application();
        app.setId(appId);
        return app;
    }
}

```
