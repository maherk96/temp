```java

@Test
void shouldReturnDashboards_whenMultipleDashboardTypesArePassed() {
    // Arrange
    Pageable pageable = PageRequest.of(0, 10);
    String currentUser = "mk66440"; 
    var dashboardTypes = List.of(DashboardType.MY_DASHBOARD, DashboardType.SYSTEM_DASHBOARD);

    Dashboard myDashboard = createDashboard(1L, "My Board", "My Board Description", currentUser, false);
    Dashboard systemDashboard = createDashboard(2L, "System Board", "System Dashboard Description", null, false);
    Dashboard othersDashboard = createDashboard(3L, "Other Board", "Created by another user", "otherUser", false);
    Dashboard deletedDashboard = createDashboard(4L, "Deleted Board", "Should not appear", currentUser, true);

    Page<Dashboard> dashboardPage =
            new PageImpl<>(List.of(myDashboard, systemDashboard, othersDashboard, deletedDashboard), pageable, 4);

    when(dashboardRepository.findAll(any(Specification.class), eq(pageable)))
            .thenReturn(dashboardPage);

    when(userManagementService.get(any(Long.class)))
            .thenReturn(new UserDTO("mk66440"));

    // Act
    Page<DashboardResponseDTO> result =
            dashboardService.findAll(null, dashboardTypes, currentUser, false, pageable);

    // Assert
    assertNotNull(result);
    assertEquals(2, result.getContent().size(), "Only MY and SYSTEM dashboards should be included");
    assertTrue(result.getContent().stream()
            .anyMatch(d -> d.dashboardName().equals("My Board")));
    assertTrue(result.getContent().stream()
            .anyMatch(d -> d.dashboardName().equals("System Board")));
    assertFalse(result.getContent().stream()
            .anyMatch(d -> d.dashboardName().equals("Other Board")));
    assertFalse(result.getContent().stream()
            .anyMatch(DashboardResponseDTO::isDeleted));

    verify(dashboardRepository, times(1)).findAll(any(Specification.class), eq(pageable));
}

```
