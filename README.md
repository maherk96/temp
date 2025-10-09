```java
@Test
void getDashboards_shouldFilterMultipleDashboardTypes() throws Exception {
    // Arrange
    int page = 1;
    int size = 10;

    // Simulate multiple dashboard types (user + system dashboards)
    var dashboardTypes = List.of(DashboardType.MY_DASHBOARD, DashboardType.SYSTEM_DASHBOARD);

    var pageable = PageRequest.of(page - 1, size);

    var dashboards = List.of(
            createResponseDTO(1L, "My Dash", false),      // mk66440’s dashboard
            createResponseDTO(2L, "System Dash", false)   // system dashboard (no user)
    );

    var dashboardPage = createPage(dashboards, pageable);

    when(dashboardManagementService.findAll(
            isNull(), eq(dashboardTypes), anyString(), eq(false), eq(pageable)))
            .thenReturn(dashboardPage);

    // Act
    mockMvc.perform(get("/api/dashboard")
                    .param("dashboardType", dashboardTypes.stream()
                            .map(Enum::name)
                            .toArray(String[]::new))
                    .param("page", String.valueOf(page))
                    .param("size", String.valueOf(size)))
            // Assert
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data.length()").value(2))
            .andExpect(jsonPath("$.data[0].dashboardName").value("My Dash"))
            .andExpect(jsonPath("$.data[1].dashboardName").value("System Dash"))
            .andExpect(jsonPath("$.total").value(2));

    // Verify
    verify(dashboardManagementService, times(1))
            .findAll(isNull(), eq(dashboardTypes), anyString(), eq(false), eq(pageable));
}

```
