```java
@AutoConfigureMockMvc
@WebMvcTest(DashboardController.class)
@ExtendWith(MockitoExtension.class)
class DashboardControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private DashboardManagementService dashboardManagementService;

    // ✅ Helper factory method for mock DTOs
    private DashboardResponseDTO createResponseDTO(long id, String name, boolean deleted) {
        return new DashboardResponseDTO(id, "mk66440", name, "Description for " + name, deleted);
    }

    private Page<DashboardResponseDTO> createPage(List<DashboardResponseDTO> dashboards, Pageable pageable) {
        return new PageImpl<>(dashboards, pageable, dashboards.size());
    }

    // ✅ 1. Default paginated response
    @Test
    void getDashboards_shouldReturnPaginatedResponse() throws Exception {
        int page = 1, size = 10;
        var pageable = PageRequest.of(page - 1, size);
        var dashboards = List.of(
                createResponseDTO(1L, "Dash 1", false),
                createResponseDTO(2L, "Dash 2", false)
        );
        var dashboardPage = createPage(dashboards, pageable);

        when(dashboardManagementService.findAll(
                isNull(), isNull(), anyString(), eq(false), eq(pageable)))
                .thenReturn(dashboardPage);

        mockMvc.perform(get("/api/dashboard")
                        .param("page", String.valueOf(page))
                        .param("size", String.valueOf(size))
                        .accept(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data.length()").value(2))
                .andExpect(jsonPath("$.data[0].dashboardName").value("Dash 1"))
                .andExpect(jsonPath("$.data[0].isDeleted").value(false))
                .andExpect(jsonPath("$.data[1].dashboardName").value("Dash 2"))
                .andExpect(jsonPath("$.total").value(2));

        verify(dashboardManagementService).findAll(
                isNull(), isNull(), anyString(), eq(false), eq(pageable));
    }

    // ✅ 2. Search filter
    @Test
    void getDashboards_shouldFilterBySearchTerm() throws Exception {
        int page = 1, size = 5;
        String search = "sales";
        var pageable = PageRequest.of(page - 1, size);
        var dashboards = List.of(createResponseDTO(1L, "Sales Report", false));
        var dashboardPage = createPage(dashboards, pageable);

        when(dashboardManagementService.findAll(
                eq(search), isNull(), anyString(), eq(false), eq(pageable)))
                .thenReturn(dashboardPage);

        mockMvc.perform(get("/api/dashboard")
                        .param("page", String.valueOf(page))
                        .param("size", String.valueOf(size))
                        .param("search", search))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data[0].dashboardName").value("Sales Report"))
                .andExpect(jsonPath("$.total").value(1));

        verify(dashboardManagementService).findAll(
                eq(search), isNull(), anyString(), eq(false), eq(pageable));
    }

    // ✅ 3. DashboardType filter (MY_DASHBOARDS)
    @Test
    void getDashboards_shouldFilterMyDashboards() throws Exception {
        int page = 1, size = 10;
        String dashboardType = "MY_DASHBOARDS";
        var pageable = PageRequest.of(page - 1, size);
        var dashboards = List.of(createResponseDTO(1L, "My Dash", false));
        var dashboardPage = createPage(dashboards, pageable);

        when(dashboardManagementService.findAll(
                isNull(), eq(dashboardType), anyString(), eq(false), eq(pageable)))
                .thenReturn(dashboardPage);

        mockMvc.perform(get("/api/dashboard")
                        .param("dashboardType", dashboardType)
                        .param("page", String.valueOf(page))
                        .param("size", String.valueOf(size)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data[0].dashboardName").value("My Dash"))
                .andExpect(jsonPath("$.total").value(1));

        verify(dashboardManagementService).findAll(
                isNull(), eq(dashboardType), anyString(), eq(false), eq(pageable));
    }

    // ✅ 4. DashboardType = SYSTEM_DASHBOARDS
    @Test
    void getDashboards_shouldFilterSystemDashboards() throws Exception {
        int page = 1, size = 10;
        String dashboardType = "SYSTEM_DASHBOARDS";
        var pageable = PageRequest.of(page - 1, size);
        var dashboards = List.of(createResponseDTO(3L, "Ops Board", true));
        var dashboardPage = createPage(dashboards, pageable);

        when(dashboardManagementService.findAll(
                isNull(), eq(dashboardType), anyString(), eq(false), eq(pageable)))
                .thenReturn(dashboardPage);

        mockMvc.perform(get("/api/dashboard")
                        .param("dashboardType", dashboardType)
                        .param("page", String.valueOf(page))
                        .param("size", String.valueOf(size)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data[0].dashboardName").value("Ops Board"))
                .andExpect(jsonPath("$.data[0].isDeleted").value(true))
                .andExpect(jsonPath("$.total").value(1));

        verify(dashboardManagementService).findAll(
                isNull(), eq(dashboardType), anyString(), eq(false), eq(pageable));
    }

    // ✅ 5. Deleted = true
    @Test
    void getDashboards_shouldIncludeDeleted_whenRequested() throws Exception {
        int page = 1, size = 10;
        boolean deleted = true;
        var pageable = PageRequest.of(page - 1, size);
        var dashboards = List.of(
                createResponseDTO(1L, "Dash 1", false),
                createResponseDTO(2L, "Deleted Dash", true)
        );
        var dashboardPage = createPage(dashboards, pageable);

        when(dashboardManagementService.findAll(
                isNull(), isNull(), anyString(), eq(deleted), eq(pageable)))
                .thenReturn(dashboardPage);

        mockMvc.perform(get("/api/dashboard")
                        .param("deleted", "true")
                        .param("page", String.valueOf(page))
                        .param("size", String.valueOf(size)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data.length()").value(2))
                .andExpect(jsonPath("$.data[1].isDeleted").value(true))
                .andExpect(jsonPath("$.total").value(2));

        verify(dashboardManagementService).findAll(
                isNull(), isNull(), anyString(), eq(deleted), eq(pageable));
    }
}

```
