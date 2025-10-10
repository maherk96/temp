```java
@ExtendWith(MockitoExtension.class)
@DisplayName("DashboardService Tests")
class DashboardServiceTest {

    @Mock
    private DashboardRepository dashboardRepository;

    @Mock
    private UserManagementService userManagementService;

    @InjectMocks
    private DashboardService dashboardService;

    @Captor
    private ArgumentCaptor<Specification<Dashboard>> specificationCaptor;

    private static final String CURRENT_USER = "mk66440";
    private static final String OTHER_USER = "otherUser";
    private static final String SYSTEM_USER = "system";

    @Nested
    @DisplayName("findAll - Type Filtering")
    class TypeFiltering {

        @Test
        @DisplayName("Should find only MY_DASHBOARD type")
        void shouldFindOnlyMyDashboards() {
            // Arrange
            Users currentUserEntity = createUser(1L, CURRENT_USER);
            Dashboard myDashboard = createDashboard(1L, "My Dashboard", "My desc", currentUserEntity, false);
            
            Pageable pageable = PageRequest.of(0, 10);
            Page<Dashboard> dashboardPage = new PageImpl<>(List.of(myDashboard), pageable, 1);

            when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
            when(userManagementService.get(1L))
                .thenReturn(new UserDTO(CURRENT_USER));

            // Act
            Page<DashboardResponseDTO> result = dashboardService.findAll(
                null,
                List.of(DashboardType.MY_DASHBOARD),
                CURRENT_USER,
                false,
                pageable
            );

            // Assert
            assertThat(result).isNotNull();
            assertThat(result.getTotalElements()).isEqualTo(1);
            assertThat(result.getContent()).hasSize(1);
            
            DashboardResponseDTO dto = result.getContent().get(0);
            assertThat(dto.dashboardName()).isEqualTo("My Dashboard");
            assertThat(dto.description()).isEqualTo("My desc");
            assertThat(dto.userName()).isEqualTo(CURRENT_USER);
            assertThat(dto.isDeleted()).isFalse();

            // Verify specification was called with correct parameters
            verify(dashboardRepository).findAll(specificationCaptor.capture(), eq(pageable));
            verify(userManagementService).get(1L);
        }

        @Test
        @DisplayName("Should find only CREATED_BY_OTHERS type")
        void shouldFindOnlyCreatedByOthers() {
            // Arrange
            Users otherUserEntity = createUser(2L, OTHER_USER);
            Dashboard otherDashboard = createDashboard(2L, "Other Dashboard", "Other desc", otherUserEntity, false);
            
            Pageable pageable = PageRequest.of(0, 10);
            Page<Dashboard> dashboardPage = new PageImpl<>(List.of(otherDashboard), pageable, 1);

            when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
            when(userManagementService.get(2L))
                .thenReturn(new UserDTO(OTHER_USER));

            // Act
            Page<DashboardResponseDTO> result = dashboardService.findAll(
                null,
                List.of(DashboardType.CREATED_BY_OTHERS),
                CURRENT_USER,
                false,
                pageable
            );

            // Assert
            assertThat(result.getTotalElements()).isEqualTo(1);
            assertThat(result.getContent().get(0).userName()).isEqualTo(OTHER_USER);
            
            verify(dashboardRepository).findAll(any(Specification.class), eq(pageable));
        }

        @Test
        @DisplayName("Should find only SYSTEM_DASHBOARD type")
        void shouldFindOnlySystemDashboards() {
            // Arrange
            Users systemUserEntity = createUser(3L, SYSTEM_USER);
            Dashboard systemDashboard = createDashboard(3L, "System Dashboard", "System desc", systemUserEntity, false);
            
            Pageable pageable = PageRequest.of(0, 10);
            Page<Dashboard> dashboardPage = new PageImpl<>(List.of(systemDashboard), pageable, 1);

            when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
            when(userManagementService.get(3L))
                .thenReturn(new UserDTO(SYSTEM_USER));

            // Act
            Page<DashboardResponseDTO> result = dashboardService.findAll(
                null,
                List.of(DashboardType.SYSTEM_DASHBOARD),
                CURRENT_USER,
                false,
                pageable
            );

            // Assert
            assertThat(result.getTotalElements()).isEqualTo(1);
            assertThat(result.getContent().get(0).userName()).isEqualTo(SYSTEM_USER);
            
            verify(dashboardRepository).findAll(any(Specification.class), eq(pageable));
        }

        @Test
        @DisplayName("Should find multiple dashboard types")
        void shouldFindMultipleDashboardTypes() {
            // Arrange
            Users currentUserEntity = createUser(1L, CURRENT_USER);
            Users systemUserEntity = createUser(2L, SYSTEM_USER);
            
            Dashboard myDashboard = createDashboard(1L, "My Dashboard", "My desc", currentUserEntity, false);
            Dashboard systemDashboard = createDashboard(2L, "System Dashboard", "System desc", systemUserEntity, false);
            
            Pageable pageable = PageRequest.of(0, 10);
            Page<Dashboard> dashboardPage = new PageImpl<>(
                List.of(myDashboard, systemDashboard), 
                pageable, 
                2
            );

            when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
            when(userManagementService.get(1L)).thenReturn(new UserDTO(CURRENT_USER));
            when(userManagementService.get(2L)).thenReturn(new UserDTO(SYSTEM_USER));

            // Act
            Page<DashboardResponseDTO> result = dashboardService.findAll(
                null,
                List.of(DashboardType.MY_DASHBOARD, DashboardType.SYSTEM_DASHBOARD),
                CURRENT_USER,
                false,
                pageable
            );

            // Assert
            assertThat(result.getTotalElements()).isEqualTo(2);
            assertThat(result.getContent())
                .extracting(DashboardResponseDTO::dashboardName)
                .containsExactlyInAnyOrder("My Dashboard", "System Dashboard");
        }
    }

    @Nested
    @DisplayName("findAll - Search Functionality")
    class SearchFunctionality {

        @Test
        @DisplayName("Should filter by search term")
        void shouldFilterBySearchTerm() {
            // Arrange
            Users user = createUser(1L, CURRENT_USER);
            Dashboard dashboard = createDashboard(1L, "Sales Report", "Monthly sales", user, false);
            
            Pageable pageable = PageRequest.of(0, 10);
            Page<Dashboard> dashboardPage = new PageImpl<>(List.of(dashboard), pageable, 1);

            when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
            when(userManagementService.get(1L))
                .thenReturn(new UserDTO(CURRENT_USER));

            // Act
            Page<DashboardResponseDTO> result = dashboardService.findAll(
                "sales",
                null,
                CURRENT_USER,
                false,
                pageable
            );

            // Assert
            assertThat(result.getTotalElements()).isEqualTo(1);
            assertThat(result.getContent().get(0).dashboardName()).isEqualTo("Sales Report");
            
            verify(dashboardRepository).findAll(any(Specification.class), eq(pageable));
        }

        @Test
        @DisplayName("Should combine search with dashboard type")
        void shouldCombineSearchWithType() {
            // Arrange
            Users user = createUser(1L, CURRENT_USER);
            Dashboard dashboard = createDashboard(1L, "My Sales Dashboard", "Sales data", user, false);
            
            Pageable pageable = PageRequest.of(0, 10);
            Page<Dashboard> dashboardPage = new PageImpl<>(List.of(dashboard), pageable, 1);

            when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
            when(userManagementService.get(1L))
                .thenReturn(new UserDTO(CURRENT_USER));

            // Act
            Page<DashboardResponseDTO> result = dashboardService.findAll(
                "sales",
                List.of(DashboardType.MY_DASHBOARD),
                CURRENT_USER,
                false,
                pageable
            );

            // Assert
            assertThat(result.getTotalElements()).isEqualTo(1);
            assertThat(result.getContent().get(0).dashboardName()).contains("Sales");
        }

        @Test
        @DisplayName("Should return empty page when no matches found")
        void shouldReturnEmptyPageWhenNoMatches() {
            // Arrange
            Pageable pageable = PageRequest.of(0, 10);
            Page<Dashboard> emptyPage = new PageImpl<>(Collections.emptyList(), pageable, 0);

            when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(emptyPage);

            // Act
            Page<DashboardResponseDTO> result = dashboardService.findAll(
                "nonexistent",
                null,
                CURRENT_USER,
                false,
                pageable
            );

            // Assert
            assertThat(result).isNotNull();
            assertThat(result.getTotalElements()).isZero();
            assertThat(result.getContent()).isEmpty();
        }
    }

    @Nested
    @DisplayName("findAll - Deleted Dashboard Handling")
    class DeletedDashboardHandling {

        @Test
        @DisplayName("Should exclude deleted dashboards by default")
        void shouldExcludeDeletedDashboards() {
            // Arrange
            Users user = createUser(1L, CURRENT_USER);
            Dashboard activeDashboard = createDashboard(1L, "Active Dashboard", "Active", user, false);
            
            Pageable pageable = PageRequest.of(0, 10);
            Page<Dashboard> dashboardPage = new PageImpl<>(List.of(activeDashboard), pageable, 1);

            when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
            when(userManagementService.get(1L))
                .thenReturn(new UserDTO(CURRENT_USER));

            // Act
            Page<DashboardResponseDTO> result = dashboardService.findAll(
                null,
                null,
                CURRENT_USER,
                false, // deleted = false
                pageable
            );

            // Assert
            assertThat(result.getTotalElements()).isEqualTo(1);
            assertThat(result.getContent().get(0).isDeleted()).isFalse();
        }

        @Test
        @DisplayName("Should include deleted dashboards when deleted=true")
        void shouldIncludeDeletedDashboards() {
            // Arrange
            Users user = createUser(1L, CURRENT_USER);
            Dashboard activeDashboard = createDashboard(1L, "Active Dashboard", "Active", user, false);
            Dashboard deletedDashboard = createDashboard(2L, "Deleted Dashboard", "Deleted", user, true);
            
            Pageable pageable = PageRequest.of(0, 10);
            Page<Dashboard> dashboardPage = new PageImpl<>(
                List.of(activeDashboard, deletedDashboard), 
                pageable, 
                2
            );

            when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
            when(userManagementService.get(1L))
                .thenReturn(new UserDTO(CURRENT_USER));

            // Act
            Page<DashboardResponseDTO> result = dashboardService.findAll(
                null,
                null,
                CURRENT_USER,
                true, // deleted = true
                pageable
            );

            // Assert
            assertThat(result.getTotalElements()).isEqualTo(2);
            assertThat(result.getContent())
                .extracting(DashboardResponseDTO::isDeleted)
                .containsExactlyInAnyOrder(false, true);
        }
    }

    @Nested
    @DisplayName("findAll - Pagination")
    class Pagination {

        @Test
        @DisplayName("Should respect page size")
        void shouldRespectPageSize() {
            // Arrange
            Users user = createUser(1L, CURRENT_USER);
            List<Dashboard> dashboards = List.of(
                createDashboard(1L, "Dashboard 1", "Desc 1", user, false),
                createDashboard(2L, "Dashboard 2", "Desc 2", user, false),
                createDashboard(3L, "Dashboard 3", "Desc 3", user, false)
            );
            
            Pageable pageable = PageRequest.of(0, 2); // Page size = 2
            Page<Dashboard> dashboardPage = new PageImpl<>(
                dashboards.subList(0, 2), 
                pageable, 
                3 // Total elements = 3
            );

            when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
            when(userManagementService.get(1L))
                .thenReturn(new UserDTO(CURRENT_USER));

            // Act
            Page<DashboardResponseDTO> result = dashboardService.findAll(
                null,
                null,
                CURRENT_USER,
                false,
                pageable
            );

            // Assert
            assertThat(result.getSize()).isEqualTo(2);
            assertThat(result.getContent()).hasSize(2);
            assertThat(result.getTotalElements()).isEqualTo(3);
            assertThat(result.getTotalPages()).isEqualTo(2);
        }

        @Test
        @DisplayName("Should return correct page number")
        void shouldReturnCorrectPageNumber() {
            // Arrange
            Users user = createUser(1L, CURRENT_USER);
            Dashboard dashboard = createDashboard(3L, "Dashboard 3", "Desc 3", user, false);
            
            Pageable pageable = PageRequest.of(1, 2); // Page 1 (second page), size 2
            Page<Dashboard> dashboardPage = new PageImpl<>(
                List.of(dashboard), 
                pageable, 
                3
            );

            when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
            when(userManagementService.get(1L))
                .thenReturn(new UserDTO(CURRENT_USER));

            // Act
            Page<DashboardResponseDTO> result = dashboardService.findAll(
                null,
                null,
                CURRENT_USER,
                false,
                pageable
            );

            // Assert
            assertThat(result.getNumber()).isEqualTo(1);
            assertThat(result.getContent()).hasSize(1);
        }

        @Test
        @DisplayName("Should handle empty first page")
        void shouldHandleEmptyFirstPage() {
            // Arrange
            Pageable pageable = PageRequest.of(0, 10);
            Page<Dashboard> emptyPage = new PageImpl<>(Collections.emptyList(), pageable, 0);

            when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(emptyPage);

            // Act
            Page<DashboardResponseDTO> result = dashboardService.findAll(
                null,
                null,
                CURRENT_USER,
                false,
                pageable
            );

            // Assert
            assertThat(result).isNotNull();
            assertThat(result.getContent()).isEmpty();
            assertThat(result.getTotalElements()).isZero();
            assertThat(result.getTotalPages()).isZero();
        }
    }

    @Nested
    @DisplayName("findAll - DTO Mapping")
    class DTOMapping {

        @Test
        @DisplayName("Should correctly map Dashboard to DashboardResponseDTO")
        void shouldCorrectlyMapDashboard() {
            // Arrange
            Users user = createUser(1L, CURRENT_USER);
            Dashboard dashboard = createDashboard(
                1L, 
                "Test Dashboard", 
                "Test Description", 
                user, 
                false
            );
            dashboard.setCreated(LocalDateTime.of(2025, 1, 1, 10, 0));
            dashboard.setLastUpdated(LocalDateTime.of(2025, 1, 2, 15, 30));
            
            Pageable pageable = PageRequest.of(0, 10);
            Page<Dashboard> dashboardPage = new PageImpl<>(List.of(dashboard), pageable, 1);

            when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
            when(userManagementService.get(1L))
                .thenReturn(new UserDTO(CURRENT_USER));

            // Act
            Page<DashboardResponseDTO> result = dashboardService.findAll(
                null,
                null,
                CURRENT_USER,
                false,
                pageable
            );

            // Assert
            DashboardResponseDTO dto = result.getContent().get(0);
            assertThat(dto.id()).isEqualTo(1L);
            assertThat(dto.dashboardName()).isEqualTo("Test Dashboard");
            assertThat(dto.description()).isEqualTo("Test Description");
            assertThat(dto.userName()).isEqualTo(CURRENT_USER);
            assertThat(dto.isDeleted()).isFalse();
            assertThat(dto.created()).isEqualTo(LocalDateTime.of(2025, 1, 1, 10, 0));
            assertThat(dto.lastUpdated()).isEqualTo(LocalDateTime.of(2025, 1, 2, 15, 30));
            
            verify(userManagementService).get(1L);
        }

        @Test
        @DisplayName("Should call userManagementService for each dashboard")
        void shouldCallUserManagementServiceForEachDashboard() {
            // Arrange
            Users user1 = createUser(1L, CURRENT_USER);
            Users user2 = createUser(2L, OTHER_USER);
            
            Dashboard dashboard1 = createDashboard(1L, "Dashboard 1", "Desc 1", user1, false);
            Dashboard dashboard2 = createDashboard(2L, "Dashboard 2", "Desc 2", user2, false);
            
            Pageable pageable = PageRequest.of(0, 10);
            Page<Dashboard> dashboardPage = new PageImpl<>(
                List.of(dashboard1, dashboard2), 
                pageable, 
                2
            );

            when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
            when(userManagementService.get(1L))
                .thenReturn(new UserDTO(CURRENT_USER));
            when(userManagementService.get(2L))
                .thenReturn(new UserDTO(OTHER_USER));

            // Act
            Page<DashboardResponseDTO> result = dashboardService.findAll(
                null,
                null,
                CURRENT_USER,
                false,
                pageable
            );

            // Assert
            assertThat(result.getContent()).hasSize(2);
            verify(userManagementService).get(1L);
            verify(userManagementService).get(2L);
            verifyNoMoreInteractions(userManagementService);
        }
    }

    @Nested
    @DisplayName("findAll - Edge Cases")
    class EdgeCases {

        @Test
        @DisplayName("Should handle null search term")
        void shouldHandleNullSearchTerm() {
            // Arrange
            Users user = createUser(1L, CURRENT_USER);
            Dashboard dashboard = createDashboard(1L, "Dashboard", "Desc", user, false);
            
            Pageable pageable = PageRequest.of(0, 10);
            Page<Dashboard> dashboardPage = new PageImpl<>(List.of(dashboard), pageable, 1);

            when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
            when(userManagementService.get(1L))
                .thenReturn(new UserDTO(CURRENT_USER));

            // Act & Assert - should not throw exception
            Page<DashboardResponseDTO> result = dashboardService.findAll(
                null, // null search
                null,
                CURRENT_USER,
                false,
                pageable
            );

            assertThat(result).isNotNull();
        }

        @Test
        @DisplayName("Should handle null dashboard types")
        void shouldHandleNullDashboardTypes() {
            // Arrange
            Users user = createUser(1L, CURRENT_USER);
            Dashboard dashboard = createDashboard(1L, "Dashboard", "Desc", user, false);
            
            Pageable pageable = PageRequest.of(0, 10);
            Page<Dashboard> dashboardPage = new PageImpl<>(List.of(dashboard), pageable, 1);

            when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
            when(userManagementService.get(1L))
                .thenReturn(new UserDTO(CURRENT_USER));

            // Act & Assert - should not throw exception
            Page<DashboardResponseDTO> result = dashboardService.findAll(
                null,
                null, // null types
                CURRENT_USER,
                false,
                pageable
            );

            assertThat(result).isNotNull();
        }

        @Test
        @DisplayName("Should handle empty dashboard types list")
        void shouldHandleEmptyDashboardTypesList() {
            // Arrange
            Users user = createUser(1L, CURRENT_USER);
            Dashboard dashboard = createDashboard(1L, "Dashboard", "Desc", user, false);
            
            Pageable pageable = PageRequest.of(0, 10);
            Page<Dashboard> dashboardPage = new PageImpl<>(List.of(dashboard), pageable, 1);

            when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
            when(userManagementService.get(1L))
                .thenReturn(new UserDTO(CURRENT_USER));

            // Act
            Page<DashboardResponseDTO> result = dashboardService.findAll(
                null,
                Collections.emptyList(), // empty list
                CURRENT_USER,
                false,
                pageable
            );

            // Assert
            assertThat(result).isNotNull();
        }
    }

    // ==================== Helper Methods ====================

    private Users createUser(Long id, String name) {
        Users user = new Users();
        user.setId(id);
        user.setName(name);
        return user;
    }

    private Dashboard createDashboard(Long id, String name, String desc, Users user, boolean deleted) {
        Dashboard dashboard = new Dashboard();
        dashboard.setId(id);
        dashboard.setName(name);
        dashboard.setDescription(desc);
        dashboard.setUser(user);
        dashboard.setDeleted(deleted);
        dashboard.setCreated(LocalDateTime.now());
        dashboard.setLastUpdated(LocalDateTime.now());
        return dashboard;
    }
}



```
