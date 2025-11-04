```java

package com.citi.fx.qa.qap.db.services;

import com.citi.fx.qa.qap.db.domain.Application;
import com.citi.fx.qa.qap.db.domain.Tag;
import com.citi.fx.qa.qap.db.dto.TagDTO;
import com.citi.fx.qa.qap.db.mappers.TagMapper;
import com.citi.fx.qa.qap.db.repositories.TagRepository;
import com.citi.fx.qa.qap.exceptions.NotFoundException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.*;
import org.springframework.data.domain.Sort;

import java.util.List;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class TagServiceTest {

    @Mock private TagRepository tagRepository;
    @Mock private ApplicationManagementService appService;
    @Mock private TagMapper tagMapper;

    @InjectMocks private TagService tagService;

    private Tag tag;
    private TagDTO tagDTO;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        tag = new Tag();
        tag.setId(1L);
        tag.setName("Regression");

        tagDTO = new TagDTO();
        tagDTO.setId(1L);
        tagDTO.setName("Regression");
    }

    // ---------------------------------------------------------------------
    // findAll()
    // ---------------------------------------------------------------------
    @Test
    void findAll_shouldReturnMappedTagList() {
        // Given
        when(tagRepository.findAll(Sort.by("id"))).thenReturn(List.of(tag));
        when(tagMapper.toDTOLIst(anyList())).thenReturn(List.of(tagDTO));

        // When
        List<TagDTO> result = tagService.findAll();

        // Then
        assertEquals(1, result.size());
        assertEquals("Regression", result.get(0).getName());
        verify(tagRepository).findAll(Sort.by("id"));
        verify(tagMapper).toDTOLIst(List.of(tag));
    }

    // ---------------------------------------------------------------------
    // findById()
    // ---------------------------------------------------------------------
    @Test
    void findById_shouldReturnMappedTagDTO() {
        // Given
        when(tagRepository.findById(1L)).thenReturn(Optional.of(tag));
        when(tagMapper.toDTO(tag)).thenReturn(tagDTO);

        // When
        TagDTO result = tagService.findById(1L);

        // Then
        assertEquals(1L, result.getId());
        assertEquals("Regression", result.getName());
        verify(tagRepository).findById(1L);
        verify(tagMapper).toDTO(tag);
    }

    @Test
    void findById_shouldThrowExceptionWhenNotFound() {
        when(tagRepository.findById(999L)).thenReturn(Optional.empty());

        assertThrows(NotFoundException.class, () -> tagService.findById(999L));
        verify(tagRepository).findById(999L);
    }

    // ---------------------------------------------------------------------
    // findEntityById()
    // ---------------------------------------------------------------------
    @Test
    void findEntityById_shouldReturnEntity() {
        when(tagRepository.findById(1L)).thenReturn(Optional.of(tag));

        Tag result = tagService.findEntityById(1L);

        assertEquals(tag, result);
        verify(tagRepository).findById(1L);
    }

    @Test
    void findEntityById_shouldThrowExceptionWhenNotFound() {
        when(tagRepository.findById(2L)).thenReturn(Optional.empty());

        assertThrows(NotFoundException.class, () -> tagService.findEntityById(2L));
        verify(tagRepository).findById(2L);
    }

    // ---------------------------------------------------------------------
    // create()
    // ---------------------------------------------------------------------
    @Test
    void create_shouldMapAndSaveTagSuccessfully() {
        // Given
        Tag unsavedTag = new Tag();
        unsavedTag.setName("Smoke");
        unsavedTag.setApp(new Application());

        Tag savedTag = new Tag();
        savedTag.setId(5L);
        savedTag.setName("Smoke");

        TagDTO dto = new TagDTO();
        dto.setName("Smoke");
        dto.setAppId(100L);

        when(tagMapper.toEntity(dto)).thenReturn(unsavedTag);
        when(appService.findByID(eq(100L), any())).thenReturn(new Application());
        when(tagRepository.save(unsavedTag)).thenReturn(savedTag);
        when(tagMapper.toDTO(savedTag)).thenReturn(new TagDTO(5L, "Smoke", 100L, null));

        // When
        TagDTO result = tagService.create(dto);

        // Then
        assertEquals(5L, result.getId());
        assertEquals("Smoke", result.getName());
        verify(tagMapper).toEntity(dto);
        verify(tagRepository).save(unsavedTag);
        verify(tagMapper).toDTO(savedTag);
    }

    // ---------------------------------------------------------------------
    // getOrCreateTag()
    // ---------------------------------------------------------------------
    @Test
    void getOrCreateTag_shouldReturnExistingTag() {
        // Given
        String tagName = "Regression";
        long appId = 50L;
        when(tagRepository.findByNameIgnoreCase(tagName)).thenReturn(Optional.of(tag));
        when(tagMapper.toDTO(tag)).thenReturn(tagDTO);

        // When
        TagDTO result = tagService.getOrCreateTag(tagName, appId);

        // Then
        assertEquals(tagDTO, result);
        verify(tagRepository).findByNameIgnoreCase(tagName);
        verify(tagMapper).toDTO(tag);
        verify(tagRepository, never()).save(any());
    }

    @Test
    void getOrCreateTag_shouldCreateNewTagWhenNotFound() {
        // Given
        String tagName = "Smoke";
        long appId = 123L;
        TagDTO newDTO = new TagDTO();
        newDTO.setName(tagName);
        newDTO.setAppId(appId);

        when(tagRepository.findByNameIgnoreCase(tagName)).thenReturn(Optional.empty());
        mockStatic(TagDTO.class);
        when(TagDTO.createTag(tagName, appId)).thenReturn(newDTO);
        when(tagMapper.toEntity(newDTO)).thenReturn(tag);
        when(tagRepository.save(tag)).thenReturn(tag);
        when(tagMapper.toDTO(tag)).thenReturn(tagDTO);

        // When
        TagDTO result = tagService.getOrCreateTag(tagName, appId);

        // Then
        assertNotNull(result);
        assertEquals("Regression", result.getName());
        verify(tagRepository).findByNameIgnoreCase(tagName);
        verify(tagRepository).save(tag);
    }
}


```
