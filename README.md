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
    void shouldReturnNonDeletedDashboards_whenDeletedFalse() {
        Pageable pageable = PageRequest.of(0, 10);
        String currentUser = "user1";
        String dashboardType = null;
        boolean deleted = false;

        Page<Dashboard> dashboardPage = new PageImpl<>(List.of(dashboard1, dashboard2), pageable, 2);

        when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);

        when(userManagementService.get(any()))
                .thenReturn(new UserDTO("User One"));

        Page<DashboardResponseDTO> result =
                dashboardService.findAll(null, dashboardType, currentUser, deleted, pageable);

        assertNotNull(result);
        assertEquals(2, result.getTotalElements());
        assertEquals("Dash 1", result.getContent().get(0).getName());
        verify(dashboardRepository).findAll(any(Specification.class), eq(pageable));
    }

    @Test
    void shouldReturnOnlyMyDashboards_whenTypeIsMyDashboards() {
        Pageable pageable = PageRequest.of(0, 10);
        String currentUser = "user1";
        String dashboardType = "MY_DASHBOARDS";

        Page<Dashboard> dashboardPage = new PageImpl<>(List.of(dashboard1), pageable, 1);

        when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
        when(userManagementService.get(any()))
                .thenReturn(new UserDTO("User One"));

        Page<DashboardResponseDTO> result =
                dashboardService.findAll(null, dashboardType, currentUser, false, pageable);

        assertEquals(1, result.getContent().size());
        assertEquals("Dash 1", result.getContent().get(0).getName());
        verify(dashboardRepository).findAll(any(Specification.class), eq(pageable));
    }

    @Test
    void shouldReturnDashboardsCreatedByOthers_whenTypeIsCreatedByOthers() {
        Pageable pageable = PageRequest.of(0, 10);
        String currentUser = "user1";
        String dashboardType = "CREATED_BY_OTHERS";

        Page<Dashboard> dashboardPage = new PageImpl<>(List.of(dashboard2), pageable, 1);

        when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
        when(userManagementService.get(any()))
                .thenReturn(new UserDTO("User Two"));

        Page<DashboardResponseDTO> result =
                dashboardService.findAll(null, dashboardType, currentUser, false, pageable);

        assertEquals(1, result.getContent().size());
        assertEquals("Sales Report", result.getContent().get(0).getName());
        verify(dashboardRepository).findAll(any(Specification.class), eq(pageable));
    }

    @Test
    void shouldReturnSystemDashboards_whenTypeIsSystemDashboards() {
        Pageable pageable = PageRequest.of(0, 10);
        String currentUser = "user1";
        String dashboardType = "SYSTEM_DASHBOARDS";

        Page<Dashboard> dashboardPage = new PageImpl<>(List.of(dashboard3), pageable, 1);

        when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
                .thenReturn(dashboardPage);
        when(userManagementService.get(any()))
                .thenReturn(new UserDTO("System Dashboard"));

        Page<DashboardResponseDTO> result =
                dashboardService.findAll(null, dashboardType, currentUser, false, pageable);

        assertEquals(1, result.getContent().size());
        assertEquals("Ops Board", result.getContent().get(0).getName());
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
        when(userManagementService.get(any()))
                .thenReturn(new UserDTO("User One"));

        Page<DashboardResponseDTO> result =
                dashboardService.findAll(null, null, currentUser, deleted, pageable);

        assertEquals(2, result.getContent().size());
        assertTrue(result.getContent().stream().anyMatch(DashboardResponseDTO::getDeleted));
        verify(dashboardRepository).findAll(any(Specification.class), eq(pageable));
    }

    // helper method
    private Dashboard createDashboard(Long id, String name, String desc, String user, boolean deleted) {
        Dashboard d = new Dashboard();
        d.setId(id);
        d.setName(name);
        d.setDescription(desc);
        d.setUser(user);
        d.setDeleted(deleted);
        return d;
    }
}
```
