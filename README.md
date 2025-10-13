```java
package com.yourpackage.mappers;

import com.yourpackage.dto.TestLaunchDTO;
import com.yourpackage.entities.*;
import com.yourpackage.exceptions.NotFoundException;
import com.yourpackage.repositories.ApplicationRepository;
import com.yourpackage.repositories.TestFeatureRepository;
import com.yourpackage.services.UserManagementService;
import com.yourpackage.services.cached.EnvironmentCachedService;
import com.yourpackage.services.cached.TestClassCachedService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mapstruct.factory.Mappers;
import org.mockito.Mockito;

import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class TestLaunchMapperTest {

    private TestLaunchMapper mapper;

    private ApplicationRepository applicationRepository;
    private TestFeatureRepository testFeatureRepository;
    private UserManagementService userService;
    private EnvironmentCachedService environmentCachedService;
    private TestClassCachedService testClassCachedService;

    @BeforeEach
    void setUp() {
        // Mock dependencies
        applicationRepository = mock(ApplicationRepository.class);
        testFeatureRepository = mock(TestFeatureRepository.class);
        userService = mock(UserManagementService.class);
        environmentCachedService = mock(EnvironmentCachedService.class);
        testClassCachedService = mock(TestClassCachedService.class);

        // Get the mapper instance
        mapper = Mappers.getMapper(TestLaunchMapper.class);

        // Inject mocks into abstract mapper
        mapper.applicationRepository = applicationRepository;
        mapper.testFeatureRepository = testFeatureRepository;
        mapper.userService = userService;
        mapper.environmentCachedService = environmentCachedService;
        mapper.testClassCachedService = testClassCachedService;
    }

    // ---------------------------------------------------------
    // DTO -> Entity tests
    // ---------------------------------------------------------

    @Test
    void shouldMapDtoToEntityWithValidReferences() {
        // Arrange
        TestLaunchDTO dto = new TestLaunchDTO();
        dto.setApp(1L);
        dto.setEnv(2L);
        dto.setTestClass(3L);
        dto.setTestFeature(4L);
        dto.setUser(5L);

        Application app = new Application(); app.setId(1L);
        Environment env = new Environment(); env.setId(2L);
        TestClass testClass = new TestClass(); testClass.setId(3L);
        TestFeature feature = new TestFeature(); feature.setId(4L);
        User user = new User(); user.setId(5L);

        when(applicationRepository.findById(1L)).thenReturn(Optional.of(app));
        when(environmentCachedService.getEnvironmentById(2L)).thenReturn(env);
        when(testClassCachedService.getTestClassById(3L)).thenReturn(testClass);
        when(testFeatureRepository.findById(4L)).thenReturn(Optional.of(feature));
        when(userService.getUserById(5L)).thenReturn(user);

        // Act
        TestLaunch entity = mapper.toEntity(dto);

        // Assert
        assertEquals(app, entity.getApp());
        assertEquals(env, entity.getEnvironment());
        assertEquals(testClass, entity.getTestClass());
        assertEquals(feature, entity.getTestFeature());
        assertEquals(user, entity.getUser());
    }

    @Test
    void shouldMapDtoToEntityWithNullReferences() {
        // Arrange
        TestLaunchDTO dto = new TestLaunchDTO();

        // Act
        TestLaunch entity = mapper.toEntity(dto);

        // Assert
        assertNull(entity.getApp());
        assertNull(entity.getEnvironment());
        assertNull(entity.getTestClass());
        assertNull(entity.getTestFeature());
        assertNull(entity.getUser());
    }

    @Test
    void shouldThrowWhenApplicationNotFound() {
        // Arrange
        TestLaunchDTO dto = new TestLaunchDTO();
        dto.setApp(99L);
        when(applicationRepository.findById(99L)).thenReturn(Optional.empty());

        // Act + Assert
        assertThrows(NotFoundException.class, () -> mapper.toEntity(dto));
    }

    // ---------------------------------------------------------
    // Entity -> DTO tests
    // ---------------------------------------------------------

    @Test
    void shouldMapEntityToDtoCorrectly() {
        // Arrange
        Application app = new Application(); app.setId(10L);
        Environment env = new Environment(); env.setId(20L);
        TestClass testClass = new TestClass(); testClass.setId(30L);
        TestFeature feature = new TestFeature(); feature.setId(40L);
        User user = new User(); user.setId(50L);

        TestLaunch entity = new TestLaunch();
        entity.setApp(app);
        entity.setEnvironment(env);
        entity.setTestClass(testClass);
        entity.setTestFeature(feature);
        entity.setUser(user);

        // Act
        TestLaunchDTO dto = mapper.toDTO(entity);

        // Assert
        assertEquals(10L, dto.getApp());
        assertEquals(20L, dto.getEnv());
        assertEquals(30L, dto.getTestClass());
        assertEquals(40L, dto.getTestFeature());
        assertEquals(50L, dto.getUser());
    }
}


```
