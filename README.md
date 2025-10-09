```java
@ExtendWith(MockitoExtension.class)
class DashboardServiceTest {

    @Mock
    private DashboardRepository dashboardRepository;

    @Mock
    private UserManagementService userManagementService;

    @InjectMocks
    private DashboardService dashboardService;

    private Dashboard dashboard1;
    private Dashboard dashboard2;
    private Dashboard dashboard3;

    @BeforeEach
    void setup() {
        dashboard1 = createDashboard(1L, "Dash 1", "Desc 1", "user1", false);
        dashboard2 = createDashboard(2L, "Sales Report", "Monthly Report", "user2", false);
        dashboard3 = createDashboard(3L, "Ops Board", "Operations", null, true);
    }

    @Test
    void shouldReturnOnlyNonDeletedDashboards_whenDeletedFalse() {
        // Arrange
        Pageable pageable = PageRequest.of(0, 10);
        String currentUser = "user1";
        String search = "";
        String dashboardType = null;
        boolean deleted = false;

        Page<Dashboard> dashboardPage = new PageImpl<>(List.of(dashboard1, dashboard2), pageable, 2);

        when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
        when(userManagementService.get(any())).thenReturn(new User("user1", "User One"));

        // Act
        Page<DashboardResponseDTO> result =
                dashboardService.findAll(search, dashboardType, currentUser, deleted, pageable);

        // Assert
        assertNotNull(result);
        assertEquals(2, result.getTotalElements());
        verify(dashboardRepository, times(1)).findAll(any(Specification.class), eq(pageable));
    }

    @Test
    void shouldFilterMyDashboards_whenTypeIsMyDashboards() {
        Pageable pageable = PageRequest.of(0, 10);
        String currentUser = "user1";
        String dashboardType = "MY_DASHBOARDS";
        boolean deleted = false;

        Page<Dashboard> dashboardPage = new PageImpl<>(List.of(dashboard1), pageable, 1);

        when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
        when(userManagementService.get(any())).thenReturn(new User("user1", "User One"));

        Page<DashboardResponseDTO> result =
                dashboardService.findAll(null, dashboardType, currentUser, deleted, pageable);

        assertEquals(1, result.getContent().size());
        assertEquals("Dash 1", result.getContent().get(0).getName());
        verify(dashboardRepository).findAll(any(Specification.class), eq(pageable));
    }

    @Test
    void shouldFilterCreatedByOthers_whenTypeIsCreatedByOthers() {
        Pageable pageable = PageRequest.of(0, 10);
        String currentUser = "user1";
        String dashboardType = "CREATED_BY_OTHERS";

        Page<Dashboard> dashboardPage = new PageImpl<>(List.of(dashboard2), pageable, 1);

        when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
        when(userManagementService.get(any())).thenReturn(new User("user2", "User Two"));

        Page<DashboardResponseDTO> result =
                dashboardService.findAll(null, dashboardType, currentUser, false, pageable);

        assertEquals(1, result.getContent().size());
        assertEquals("Sales Report", result.getContent().get(0).getName());
        verify(dashboardRepository).findAll(any(Specification.class), eq(pageable));
    }

    @Test
    void shouldFilterSystemDashboards_whenTypeIsSystemDashboards() {
        Pageable pageable = PageRequest.of(0, 10);
        String currentUser = "user1";
        String dashboardType = "SYSTEM_DASHBOARDS";

        Page<Dashboard> dashboardPage = new PageImpl<>(List.of(dashboard3), pageable, 1);

        when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
        when(userManagementService.get(any())).thenReturn(new User("system", "System Dashboard"));

        Page<DashboardResponseDTO> result =
                dashboardService.findAll(null, dashboardType, currentUser, false, pageable);

        assertEquals(1, result.getContent().size());
        assertTrue(result.getContent().get(0).getDeleted());
        verify(dashboardRepository).findAll(any(Specification.class), eq(pageable));
    }

    @Test
    void shouldIncludeDeletedDashboards_whenDeletedTrue() {
        Pageable pageable = PageRequest.of(0, 10);
        String currentUser = "user1";
        boolean deleted = true;

        Page<Dashboard> dashboardPage = new PageImpl<>(List.of(dashboard1, dashboard3), pageable, 2);

        when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
        when(userManagementService.get(any())).thenReturn(new User("user1", "User One"));

        Page<DashboardResponseDTO> result =
                dashboardService.findAll(null, null, currentUser, deleted, pageable);

        assertEquals(2, result.getContent().size());
        assertTrue(result.getContent().stream().anyMatch(DashboardResponseDTO::getDeleted));
        verify(dashboardRepository).findAll(any(Specification.class), eq(pageable));
    }

    // helper factory
    private Dashboard createDashboard(Long id, String name, String desc, String user, boolean deleted) {
        Dashboard d = new Dashboard();
        d.setId(id);
        d.setName(name);
        d.setDescription(desc);
        d.setUser(user);
        d.setDeleted(deleted);
        return d;
    }

    // mock user object for userManagementService
    static class User {
        private final String id;
        private final String name;

        User(String id, String name) {
            this.id = id;
            this.name = name;
        }

        public String getName() {
            return name;
        }
    }
}
```
