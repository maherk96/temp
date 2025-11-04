```java

@Test
void getOrCreateTag_shouldCreateNewTagWhenNotFound() {
    // Given
    String tagName = "Smoke";
    long appId = 123L;
    Tag newTag = new Tag();

    TagDTO expectedDTO = new TagDTO();
    expectedDTO.setId(10L);
    expectedDTO.setName(tagName);
    expectedDTO.setAppId(appId);

    when(tagRepository.findByNameIgnoreCase(tagName)).thenReturn(Optional.empty());
    when(tagMapper.toEntity(any())).thenReturn(newTag);
    when(tagRepository.save(newTag)).thenReturn(newTag);
    when(tagMapper.toDTO(newTag)).thenReturn(expectedDTO);

    // When
    TagDTO result = tagService.getOrCreateTag(tagName, appId);

    // Then
    assertNotNull(result);
    assertEquals(tagName, result.getName());
    verify(tagRepository).findByNameIgnoreCase(tagName);
    verify(tagRepository).save(newTag);
}

```
