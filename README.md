```java
    @Test
    void get_existingDashboard_returnsDTO() {
        var user = new Users();
        user.setName("testUser");
        user = userRepository.save(user);

        var dash = new Dashboard();
        dash.setName("My Dashboard");
        dash.setUser(user);
        dash = dashboardRepository.save(dash);

        var dto = dashboardService.get(dash.getId());

        assertNotNull(dto);
        assertEquals(dash.getId(), dto.getId());
        assertEquals("My Dashboard", dto.getName());
    }

    @Test
    void get_nonExistingDashboard_throws() {
        assertThrows(NotFoundException.class, () -> dashboardService.get(999L));
    }

    @Test
    void create_usesUserManagementService() {
        var fakeUser = new Users();
        fakeUser.setId(42L);
        fakeUser.setName("mockUser");

        when(userManagementService.getOrCreateUser("mockUser")).thenReturn(fakeUser);

        var details = new DashboardDetails("Dash Name", "Desc", "mockUser");

        var dto = dashboardService.create(details);

        assertNotNull(dto.getId());
        assertEquals("Dash Name", dto.getName());
        assertEquals("Desc", dto.getDescription());

        verify(userManagementService, times(1)).getOrCreateUser("mockUser");
    }

    @Test
    void update_changesNameAndDescription() {
        var user = new Users();
        user.setName("testUser");
        user = userRepository.save(user);

        var dash = new Dashboard();
        dash.setName("Old Name");
        dash.setDescription("Old Desc");
        dash.setUser(user);
        dash = dashboardRepository.save(dash);

        var update = new DashboardUpdate("New Name", "New Desc");

        var dto = dashboardService.update(dash.getId(), update);

        assertEquals("New Name", dto.getName());
        assertEquals("New Desc", dto.getDescription());
    }

    @Test
    void update_nonExistingDashboard_throws() {
        var update = new DashboardUpdate("Name", "Desc");
        assertThrows(NotFoundException.class, () -> dashboardService.update(999L, update));
    }

    @Test
    void delete_marksDashboardAsDeleted() {
        var user = new Users();
        user.setName("testUser");
        user = userRepository.save(user);

        var dash = new Dashboard();
        dash.setName("To Delete");
        dash.setUser(user);
        dash = dashboardRepository.save(dash);

        dashboardService.delete(dash.getId());

        var deleted = dashboardRepository.findById(dash.getId()).orElseThrow();
        assertTrue(deleted.getDeleted());
        assertNotNull(deleted.getLastUpdate());
    }

    @Test
    void delete_nonExistingDashboard_throws() {
        assertThrows(NotFoundException.class, () -> dashboardService.delete(999L));
    }
```
