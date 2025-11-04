```java


package com.citi.fx.qa.qap.db.services;

import com.citi.fx.qa.qap.db.config.CacheConfig;
import com.citi.fx.qa.qap.db.dto.TagDTO;
import com.citi.fx.qa.qap.db.entities.Tag;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.test.annotation.DirtiesContext;
import org.springframework.test.context.ContextConfiguration;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

/**
 * Integration test for verifying caching and eviction behavior
 * of {@link TagManagementService}, including findById() and findEntityById().
 */
@EnableCaching
@SpringBootTest
@ContextConfiguration(classes = {TagManagementService.class, CacheConfig.class})
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
class TagManagementServiceTest {

    @MockBean
    private TagService tagService;

    @Autowired
    private TagManagementService tagManagementService;

    @Autowired
    private CacheManager cacheManager;

    @BeforeEach
    void clearCache() {
        var cache = cacheManager.getCache("tagCache");
        if (cache != null) cache.clear();
    }

    // -------------------------------------------------------------------------
    // EXISTING TESTS
    // -------------------------------------------------------------------------

    @Test
    void testGetOrCreateTag_isCached() {
        String tagName = "Regression";
        long appId = 100L;
        TagDTO expected = new TagDTO();
        expected.setId(1L);
        expected.setName(tagName);
        expected.setAppId(appId);

        when(tagService.getOrCreateTag(tagName, appId)).thenReturn(expected);

        TagDTO firstCall = tagManagementService.getOrCreateTag(tagName, appId);
        TagDTO secondCall = tagManagementService.getOrCreateTag(tagName, appId);

        assertSame(firstCall, secondCall);
        verify(tagService, times(1)).getOrCreateTag(tagName, appId);
    }

    @Test
    void testCreate_shouldEvictCache() {
        String tagName = "NewTag";
        long appId = 200L;

        TagDTO cachedTag = new TagDTO();
        cachedTag.setId(1L);
        cachedTag.setName(tagName);
        cachedTag.setAppId(appId);

        TagDTO newTag = new TagDTO();
        newTag.setId(2L);
        newTag.setName(tagName);
        newTag.setAppId(appId);

        when(tagService.getOrCreateTag(tagName, appId)).thenReturn(cachedTag);
        tagManagementService.getOrCreateTag(tagName, appId);
        verify(tagService, times(1)).getOrCreateTag(tagName, appId);

        when(tagService.create(newTag)).thenReturn(newTag);
        long createdId = tagManagementService.create(newTag);
        assertEquals(2L, createdId);

        when(tagService.getOrCreateTag(tagName, appId)).thenReturn(newTag);
        tagManagementService.getOrCreateTag(tagName, appId);
        verify(tagService, times(2)).getOrCreateTag(tagName, appId);
    }

    @Test
    void testClearCache_shouldEvictAllEntries() {
        String tagName = "Smoke";
        long appId = 500L;
        TagDTO tag = new TagDTO();
        tag.setId(9L);
        tag.setName(tagName);
        tag.setAppId(appId);

        when(tagService.getOrCreateTag(tagName, appId)).thenReturn(tag);

        tagManagementService.getOrCreateTag(tagName, appId);
        tagManagementService.clearCache();
        tagManagementService.getOrCreateTag(tagName, appId);

        verify(tagService, times(2)).getOrCreateTag(tagName, appId);
    }

    // -------------------------------------------------------------------------
    // NEW TESTS FOR findById() and findEntityById()
    // -------------------------------------------------------------------------

    /**
     * Verifies that findById() caches the TagDTO after first retrieval.
     */
    @Test
    void testFindById_isCached() {
        Long tagId = 10L;
        TagDTO expected = new TagDTO();
        expected.setId(tagId);
        expected.setName("Regression");
        expected.setAppId(999L);

        when(tagService.findById(tagId)).thenReturn(expected);

        TagDTO firstCall = tagManagementService.findById(tagId);
        TagDTO secondCall = tagManagementService.findById(tagId);

        assertSame(firstCall, secondCall, "findById should return cached instance");
        verify(tagService, times(1)).findById(tagId);
    }

    /**
     * Verifies that findEntityById() caches the Tag entity after first retrieval.
     */
    @Test
    void testFindEntityById_isCached() {
        Long tagId = 20L;
        Tag entity = new Tag();
        entity.setId(tagId);
        entity.setName("Smoke");

        when(tagService.findEntityById(tagId)).thenReturn(entity);

        Tag firstCall = tagManagementService.findEntityById(tagId);
        Tag secondCall = tagManagementService.findEntityById(tagId);

        assertSame(firstCall, secondCall, "findEntityById should return cached entity instance");
        verify(tagService, times(1)).findEntityById(tagId);
    }

    /**
     * Verifies that cache eviction after create() invalidates cached entries from findById().
     */
    @Test
    void testCreate_shouldEvictFindByIdCache() {
        Long tagId = 30L;
        TagDTO cachedTag = new TagDTO();
        cachedTag.setId(tagId);
        cachedTag.setName("Regression");

        when(tagService.findById(tagId)).thenReturn(cachedTag);
        tagManagementService.findById(tagId);
        verify(tagService, times(1)).findById(tagId);

        TagDTO newTag = new TagDTO();
        newTag.setId(999L);
        newTag.setName("NewRegression");
        when(tagService.create(newTag)).thenReturn(newTag);
        tagManagementService.create(newTag);

        // After create, cached findById entry should be gone
        tagManagementService.findById(tagId);
        verify(tagService, times(2)).findById(tagId);
    }

    /**
     * Verifies that clearCache() also clears cached findEntityById() results.
     */
    @Test
    void testClearCache_shouldEvictFindEntityByIdCache() {
        Long tagId = 40L;
        Tag tag = new Tag();
        tag.setId(tagId);
        tag.setName("Performance");

        when(tagService.findEntityById(tagId)).thenReturn(tag);

        tagManagementService.findEntityById(tagId);
        tagManagementService.clearCache();
        tagManagementService.findEntityById(tagId);

        verify(tagService, times(2)).findEntityById(tagId);
    }
}
```
