```java
package com.citi.fx.qa.qap.db.service.user;

import com.citi.fx.qa.qap.db.domain.Users;
import com.citi.fx.qa.qap.db.dto.UserDTO;
import com.citi.fx.qa.qap.db.mappers.UserMapper;
import com.citi.fx.qa.qap.db.repositories.UserRepository;
import com.citi.fx.qa.qap.db.exceptions.NotFoundException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.MockedStatic;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository usersRepository;

    @Mock
    private UserMapper userMapper;

    @InjectMocks
    private UserService userService;

    private Users testUser;
    private UserDTO testUserDto;

    @BeforeEach
    void setup() {
        testUser = new Users();
        testUser.setId(1L);
        testUser.setName("TestUser");

        testUserDto = new UserDTO(1L, "TestUser");
    }

    // -----------------------------------------------------
    // get(id)
    // -----------------------------------------------------

    @Test
    void get_shouldReturnMappedUser_whenUserExists() {
        when(usersRepository.findById(1L)).thenReturn(Optional.of(testUser));
        when(userMapper.toDTO(testUser)).thenReturn(testUserDto);

        UserDTO result = userService.get(1L);

        assertThat(result).isEqualTo(testUserDto);
        verify(usersRepository).findById(1L);
        verify(userMapper).toDTO(testUser);
    }

    @Test
    void get_shouldThrowException_whenUserNotFound() {
        when(usersRepository.findById(999L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> userService.get(999L))
                .isInstanceOf(NotFoundException.class);
    }

    // -----------------------------------------------------
    // create()
    // -----------------------------------------------------

    @Test
    void create_shouldSaveUserAndReturnId() {
        when(userMapper.toEntity(testUserDto)).thenReturn(testUser);
        when(usersRepository.save(testUser)).thenReturn(testUser);

        Long result = userService.create(testUserDto);

        assertThat(result).isEqualTo(1L);
        verify(userMapper).toEntity(testUserDto);
        verify(usersRepository).save(testUser);
    }

    // -----------------------------------------------------
    // getOrCreateUser()
    // -----------------------------------------------------

    @Test
    void getOrCreateUser_shouldReturnExistingUserId_whenUserExists() {
        when(usersRepository.findByNameIgnoreCase("mk66441"))
                .thenReturn(Optional.of(testUser));

        long id = userService.getOrCreateUser("mk66441");

        assertThat(id).isEqualTo(1L);
        verify(usersRepository).findByNameIgnoreCase("mk66441");
        verifyNoMoreInteractions(usersRepository);
    }

    @Test
    void getOrCreateUser_shouldCreateNewUser_whenNotExists() {
        when(usersRepository.findByNameIgnoreCase("newuser")).thenReturn(Optional.empty());

        var newUserDto = new UserDTO(null, "newuser");
        var newUserEntity = new Users();
        newUserEntity.setId(99L);
        newUserEntity.setName("newuser");

        // Mock static method
        try (MockedStatic<UserDTO> mockStatic = mockStatic(UserDTO.class)) {
            mockStatic.when(() -> UserDTO.createUserWithUserName("newuser"))
                      .thenReturn(newUserDto);

            when(userMapper.toEntity(newUserDto)).thenReturn(newUserEntity);
            when(usersRepository.save(newUserEntity)).thenReturn(newUserEntity);

            long id = userService.getOrCreateUser("newuser");

            assertThat(id).isEqualTo(99L);
            verify(usersRepository).findByNameIgnoreCase("newuser");
            verify(usersRepository).save(newUserEntity);
        }
    }

    // -----------------------------------------------------
    // getUserById()
    // -----------------------------------------------------

    @Test
    void getUserById_shouldReturnUser_whenExists() {
        when(usersRepository.findById(1L)).thenReturn(Optional.of(testUser));

        Users result = userService.getUserById(1L);

        assertThat(result).isEqualTo(testUser);
        verify(usersRepository).findById(1L);
    }

    @Test
    void getUserById_shouldThrowException_whenNotFound() {
        when(usersRepository.findById(5L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> userService.getUserById(5L))
                .isInstanceOf(NotFoundException.class)
                .hasMessageContaining("User id 5 was not found");
    }
}


package com.citi.fx.qa.qap.db.service.user;

import com.citi.fx.qa.qap.db.domain.Users;
import com.citi.fx.qa.qap.db.dto.UserDTO;
import com.citi.fx.qa.qap.db.mappers.UserMapper;
import com.citi.fx.qa.qap.db.repositories.UserRepository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.context.annotation.Import;
import org.springframework.transaction.annotation.Transactional;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@DataJpaTest
@Import({UserService.class, UserMapper.class})
@Transactional
class UserServiceIntegrationTest {

    @Autowired
    private UserService userService;

    @Autowired
    private UserRepository userRepository;

    // ---------------------------------------------------------
    // create()
    // ---------------------------------------------------------

    @Test
    void create_whenValidUserDTO_shouldPersistAndReturnId() {
        UserDTO dto = new UserDTO(null, "IntegrationUser");

        Long id = userService.create(dto);

        assertThat(id).isNotNull();
        Optional<Users> saved = userRepository.findById(id);
        assertThat(saved).isPresent();
        assertThat(saved.get().getName()).isEqualTo("IntegrationUser");
    }

    // ---------------------------------------------------------
    // get()
    // ---------------------------------------------------------

    @Test
    void get_whenUserExists_shouldReturnMappedDTO() {
        Users user = new Users();
        user.setName("FetchMe");
        user = userRepository.save(user);

        UserDTO result = userService.get(user.getId());

        assertThat(result.getName()).isEqualTo("FetchMe");
        assertThat(result.getId()).isEqualTo(user.getId());
    }

    @Test
    void get_whenUserDoesNotExist_shouldThrowException() {
        assertThatThrownBy(() -> userService.get(999L))
                .hasMessageContaining("not found");
    }

    // ---------------------------------------------------------
    // getOrCreateUser()
    // ---------------------------------------------------------

    @Test
    void getOrCreateUser_whenUserNotExists_shouldCreateAndReturnNewUser() {
        long id = userService.getOrCreateUser("NewIntegrationUser");

        assertThat(id).isNotNull();
        assertThat(userRepository.count()).isEqualTo(1);
        assertThat(userRepository.findById(id))
                .isPresent()
                .get()
                .extracting(Users::getName)
                .isEqualTo("NewIntegrationUser");
    }

    @Test
    void getOrCreateUser_whenUserAlreadyExists_shouldReturnExistingId() {
        Users user = new Users();
        user.setName("ExistingIntegrationUser");
        user = userRepository.save(user);

        long id = userService.getOrCreateUser("ExistingIntegrationUser");

        assertThat(id).isEqualTo(user.getId());
        assertThat(userRepository.count()).isEqualTo(1);
    }

    // ---------------------------------------------------------
    // getUserById()
    // ---------------------------------------------------------

    @Test
    void getUserById_whenUserExists_shouldReturnEntity() {
        Users user = new Users();
        user.setName("EntityCheck");
        user = userRepository.save(user);

        Users found = userService.getUserById(user.getId());

        assertThat(found.getId()).isEqualTo(user.getId());
        assertThat(found.getName()).isEqualTo("EntityCheck");
    }

    @Test
    void getUserById_whenUserDoesNotExist_shouldThrowException() {
        assertThatThrownBy(() -> userService.getUserById(1000L))
                .hasMessageContaining("User id 1000 was not found");
    }
}



```
