```java
@DataJpaTest
@Transactional
@DisplayName("DashboardSpecification Tests")
class DashboardSpecificationTest {

    @Autowired
    private DashboardRepository dashboardRepository;

    @Autowired
    private UserRepository userRepository;

    private static final String CURRENT_USER = "mk66440";
    private static final String OTHER_USER = "otherUser";
    private static final String SYSTEM_SOEID = "system";

    // Dashboard name constants
    private static class DashboardNames {
        static final String FUSION_BOARD = "Fusion Board";
        static final String ALGO_BOARD = "Algo Board";
        static final String ANALYTICS_DASHBOARD = "Analytics Dashboard";
        static final String REGRESSION_MONITOR = "Regression Monitor";
        static final String ALERTS_OVERVIEW = "Alerts Overview";
        static final String TRADING_STATUS = "Trading Status";
        static final String OLD_DASHBOARD = "Old Dashboard";
        static final String ARCHIVED_REPORT = "Archived Report";
    }

    private Users currentUserEntity;
    private Users otherUserEntity;
    private Users systemUserEntity;

    @BeforeEach
    void setup() {
        // Create users only (dashboards created per-test)
        currentUserEntity = createAndSaveUser(CURRENT_USER);
        otherUserEntity = createAndSaveUser(OTHER_USER);
        systemUserEntity = createAndSaveUser(SYSTEM_SOEID);
    }

    // ==================== Nested Test Classes ====================

    @Nested
    @DisplayName("Dashboard Type Filtering")
    class DashboardTypeFiltering {

        @Test
        @DisplayName("Should return only current user's dashboards")
        void shouldReturnOnlyMyDashboards() {
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);
            createAndSave(DashboardNames.ANALYTICS_DASHBOARD, "Performance analytics", currentUserEntity, false);
            createAndSave(DashboardNames.TRADING_STATUS, "Current trading status", currentUserEntity, false);
            createAndSave(DashboardNames.ALGO_BOARD, "Other user's algo", otherUserEntity, false);
            createAndSave(DashboardNames.REGRESSION_MONITOR, "System default", systemUserEntity, false);

            List<String> results = getDashboardNames(null, List.of(DashboardType.MY_DASHBOARD), false);

            assertThat(results).containsExactlyInAnyOrder(
                DashboardNames.FUSION_BOARD,
                DashboardNames.ANALYTICS_DASHBOARD,
                DashboardNames.TRADING_STATUS
            );
        }

        @Test
        @DisplayName("Should return only dashboards created by others (excluding system)")
        void shouldReturnDashboardsCreatedByOthers() {
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);
            createAndSave(DashboardNames.ALGO_BOARD, "Other user's algo", otherUserEntity, false);
            createAndSave(DashboardNames.REGRESSION_MONITOR, "System default", systemUserEntity, false);

            List<String> results = getDashboardNames(null, List.of(DashboardType.CREATED_BY_OTHERS), false);

            assertThat(results).containsExactlyInAnyOrder(DashboardNames.ALGO_BOARD);
        }

        @Test
        @DisplayName("Should return only system dashboards")
        void shouldReturnSystemDashboards() {
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);
            createAndSave(DashboardNames.REGRESSION_MONITOR, "System default", systemUserEntity, false);
            createAndSave(DashboardNames.ALERTS_OVERVIEW, "System alerts", systemUserEntity, false);

            List<String> results = getDashboardNames(null, List.of(DashboardType.SYSTEM_DASHBOARD), false);

            assertThat(results).containsExactlyInAnyOrder(
                DashboardNames.REGRESSION_MONITOR,
                DashboardNames.ALERTS_OVERVIEW
            );
        }

        @Test
        @DisplayName("Should return dashboards from multiple types (MY_DASHBOARD + SYSTEM_DASHBOARD)")
        void shouldReturnMyAndSystemDashboards() {
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);
            createAndSave(DashboardNames.ANALYTICS_DASHBOARD, "Performance analytics", currentUserEntity, false);
            createAndSave(DashboardNames.ALGO_BOARD, "Other user's algo", otherUserEntity, false);
            createAndSave(DashboardNames.REGRESSION_MONITOR, "System default", systemUserEntity, false);
            createAndSave(DashboardNames.ALERTS_OVERVIEW, "System alerts", systemUserEntity, false);

            List<String> results = getDashboardNames(
                null,
                List.of(DashboardType.MY_DASHBOARD, DashboardType.SYSTEM_DASHBOARD),
                false
            );

            assertThat(results).containsExactlyInAnyOrder(
                DashboardNames.FUSION_BOARD,
                DashboardNames.ANALYTICS_DASHBOARD,
                DashboardNames.REGRESSION_MONITOR,
                DashboardNames.ALERTS_OVERVIEW
            );
        }

        @Test
        @DisplayName("Should return all non-deleted dashboards when no type filter specified")
        void shouldReturnAllNonDeletedDashboards() {
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);
            createAndSave(DashboardNames.ALGO_BOARD, "Other user's algo", otherUserEntity, false);
            createAndSave(DashboardNames.REGRESSION_MONITOR, "System default", systemUserEntity, false);
            createAndSave(DashboardNames.OLD_DASHBOARD, "Deleted", currentUserEntity, true);

            List<String> results = getDashboardNames(null, null, false);

            assertThat(results).containsExactlyInAnyOrder(
                DashboardNames.FUSION_BOARD,
                DashboardNames.ALGO_BOARD,
                DashboardNames.REGRESSION_MONITOR
            );
        }

        @Test
        @DisplayName("Should handle empty dashboard types list")
        void shouldHandleEmptyDashboardTypesList() {
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);
            createAndSave(DashboardNames.ALGO_BOARD, "Other user's algo", otherUserEntity, false);

            List<String> results = getDashboardNames(null, Collections.emptyList(), false);

            // Empty list should return all dashboards
            assertThat(results).containsExactlyInAnyOrder(
                DashboardNames.FUSION_BOARD,
                DashboardNames.ALGO_BOARD
            );
        }
    }

    @Nested
    @DisplayName("Search Functionality")
    class SearchFunctionality {

        @Test
        @DisplayName("Should match search term in dashboard name")
        void shouldMatchSearchTermInName() {
            createAndSave(DashboardNames.ALGO_BOARD, "Other user's algo", otherUserEntity, false);
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);

            List<String> results = getDashboardNames("algo", null, false);

            assertThat(results).containsExactlyInAnyOrder(DashboardNames.ALGO_BOARD);
        }

        @Test
        @DisplayName("Should match search term in dashboard description")
        void shouldMatchSearchTermInDescription() {
            createAndSave(DashboardNames.TRADING_STATUS, "Current trading status view", currentUserEntity, false);
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);
            createAndSave(DashboardNames.ALGO_BOARD, "Algorithm dashboard", otherUserEntity, false);

            List<String> results = getDashboardNames("status", null, false);

            assertThat(results).containsExactlyInAnyOrder(DashboardNames.TRADING_STATUS);
        }

        @ParameterizedTest
        @CsvSource({
            "fusion board, " + "Fusion Board",
            "ANALYTICS DASHBOARD, " + "Analytics Dashboard",
            "AlGo BoArD, " + "Algo Board"
        })
        @DisplayName("Should perform case-insensitive search")
        void shouldPerformCaseInsensitiveSearch(String searchTerm, String expectedDashboard) {
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);
            createAndSave(DashboardNames.ANALYTICS_DASHBOARD, "Performance analytics", currentUserEntity, false);
            createAndSave(DashboardNames.ALGO_BOARD, "Other user's algo", otherUserEntity, false);

            List<String> results = getDashboardNames(searchTerm, null, false);

            assertThat(results).contains(expectedDashboard);
        }

        @Test
        @DisplayName("Should match partial search terms")
        void shouldMatchPartialSearchTerms() {
            createAndSave(DashboardNames.ANALYTICS_DASHBOARD, "Performance analytics", currentUserEntity, false);
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);

            // "board" matches both "Fusion Board" and "Analytics Dashboard"
            List<String> results = getDashboardNames("board", null, false);

            assertThat(results).containsExactlyInAnyOrder(
                DashboardNames.FUSION_BOARD,
                DashboardNames.ANALYTICS_DASHBOARD
            );
        }

        @Test
        @DisplayName("Should return empty list when no dashboards match search")
        void shouldReturnEmptyListWhenNoMatch() {
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);

            List<String> results = getDashboardNames("nonexistent", null, false);

            assertThat(results).isEmpty();
        }

        @Test
        @DisplayName("Should handle empty search string as no search filter")
        void shouldHandleEmptySearchString() {
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);
            createAndSave(DashboardNames.ALGO_BOARD, "Other user's algo", otherUserEntity, false);

            List<String> results = getDashboardNames("", null, false);

            // Empty string should return all dashboards
            assertThat(results).containsExactlyInAnyOrder(
                DashboardNames.FUSION_BOARD,
                DashboardNames.ALGO_BOARD
            );
        }

        @Test
        @DisplayName("Should handle special characters in search")
        void shouldHandleSpecialCharactersInSearch() {
            createAndSave("Dashboard (Test)", "Special chars", currentUserEntity, false);
            createAndSave("Dashboard [Beta]", "More special chars", currentUserEntity, false);

            List<String> results = getDashboardNames("(test)", null, false);

            assertThat(results).containsExactlyInAnyOrder("Dashboard (Test)");
        }
    }

    @Nested
    @DisplayName("Combined Filters")
    class CombinedFilters {

        @Test
        @DisplayName("Should combine search with MY_DASHBOARD type")
        void shouldCombineSearchWithMyDashboardType() {
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);
            createAndSave(DashboardNames.ANALYTICS_DASHBOARD, "Performance analytics", currentUserEntity, false);
            createAndSave(DashboardNames.ALGO_BOARD, "Other user's algo", otherUserEntity, false);

            List<String> results = getDashboardNames("fusion", List.of(DashboardType.MY_DASHBOARD), false);

            assertThat(results).containsExactlyInAnyOrder(DashboardNames.FUSION_BOARD);
        }

        @Test
        @DisplayName("Should combine search with CREATED_BY_OTHERS type")
        void shouldCombineSearchWithCreatedByOthersType() {
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);
            createAndSave(DashboardNames.ALGO_BOARD, "Other user's algo", otherUserEntity, false);

            List<String> results = getDashboardNames("algo", List.of(DashboardType.CREATED_BY_OTHERS), false);

            assertThat(results).containsExactlyInAnyOrder(DashboardNames.ALGO_BOARD);
        }

        @Test
        @DisplayName("Should combine search with SYSTEM_DASHBOARD type")
        void shouldCombineSearchWithSystemDashboardType() {
            createAndSave(DashboardNames.REGRESSION_MONITOR, "System default", systemUserEntity, false);
            createAndSave(DashboardNames.ALERTS_OVERVIEW, "System alerts", systemUserEntity, false);
            createAndSave(DashboardNames.FUSION_BOARD, "My regression test", currentUserEntity, false);

            List<String> results = getDashboardNames("regression", List.of(DashboardType.SYSTEM_DASHBOARD), false);

            assertThat(results).containsExactlyInAnyOrder(DashboardNames.REGRESSION_MONITOR);
        }

        @Test
        @DisplayName("Should combine search with multiple dashboard types")
        void shouldCombineSearchWithMultipleTypes() {
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);
            createAndSave(DashboardNames.ANALYTICS_DASHBOARD, "Performance analytics", currentUserEntity, false);
            createAndSave(DashboardNames.ALGO_BOARD, "Other user's algo", otherUserEntity, false);
            createAndSave(DashboardNames.REGRESSION_MONITOR, "System default", systemUserEntity, false);

            List<String> results = getDashboardNames(
                "board",
                List.of(DashboardType.MY_DASHBOARD, DashboardType.CREATED_BY_OTHERS),
                false
            );

            assertThat(results).containsExactlyInAnyOrder(
                DashboardNames.FUSION_BOARD,
                DashboardNames.ANALYTICS_DASHBOARD,
                DashboardNames.ALGO_BOARD
            );
        }
    }

    @Nested
    @DisplayName("Deleted Dashboard Handling")
    class DeletedDashboardHandling {

        @Test
        @DisplayName("Should exclude deleted dashboards by default")
        void shouldExcludeDeletedDashboards() {
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);
            createAndSave(DashboardNames.OLD_DASHBOARD, "Deleted dashboard", currentUserEntity, true);

            List<String> results = getDashboardNames(null, List.of(DashboardType.MY_DASHBOARD), false);

            assertThat(results).containsExactlyInAnyOrder(DashboardNames.FUSION_BOARD);
        }

        @Test
        @DisplayName("Should include deleted dashboards when includeDeleted is true")
        void shouldIncludeDeletedDashboards() {
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);
            createAndSave(DashboardNames.OLD_DASHBOARD, "Deleted dashboard", currentUserEntity, true);
            createAndSave(DashboardNames.ANALYTICS_DASHBOARD, "Performance analytics", currentUserEntity, false);

            List<String> results = getDashboardNames(null, List.of(DashboardType.MY_DASHBOARD), true);

            assertThat(results).containsExactlyInAnyOrder(
                DashboardNames.FUSION_BOARD,
                DashboardNames.OLD_DASHBOARD,
                DashboardNames.ANALYTICS_DASHBOARD
            );
        }

        @Test
        @DisplayName("Should include deleted dashboards in search when includeDeleted is true")
        void shouldIncludeDeletedDashboardsInSearch() {
            createAndSave(DashboardNames.OLD_DASHBOARD, "Old system", currentUserEntity, true);
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);

            List<String> results = getDashboardNames("old", null, true);

            assertThat(results).containsExactlyInAnyOrder(DashboardNames.OLD_DASHBOARD);
        }

        @Test
        @DisplayName("Should return all dashboards including deleted when no filters and includeDeleted is true")
        void shouldReturnAllDashboardsIncludingDeleted() {
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);
            createAndSave(DashboardNames.ALGO_BOARD, "Other user's algo", otherUserEntity, false);
            createAndSave(DashboardNames.OLD_DASHBOARD, "Deleted", currentUserEntity, true);
            createAndSave(DashboardNames.ARCHIVED_REPORT, "Archived", otherUserEntity, true);

            List<String> results = getDashboardNames(null, null, true);

            assertThat(results).containsExactlyInAnyOrder(
                DashboardNames.FUSION_BOARD,
                DashboardNames.ALGO_BOARD,
                DashboardNames.OLD_DASHBOARD,
                DashboardNames.ARCHIVED_REPORT
            );
        }
    }

    @Nested
    @DisplayName("Edge Cases")
    class EdgeCases {

        @Test
        @DisplayName("Should return other user's dashboards when current user has none")
        void shouldReturnOthersDashboardsWhenCurrentUserHasNone() {
            Users newUser = createAndSaveUser("newUser");
            createAndSave(DashboardNames.FUSION_BOARD, "Other's dashboard", currentUserEntity, false);
            createAndSave(DashboardNames.ALGO_BOARD, "Another other's dashboard", otherUserEntity, false);

            Specification<Dashboard> spec = DashboardSpecification.build(
                null,
                List.of(DashboardType.CREATED_BY_OTHERS),
                newUser.getName(),
                false
            );

            List<Dashboard> results = dashboardRepository.findAll(spec);

            assertThat(results)
                .extracting(Dashboard::getName)
                .containsExactlyInAnyOrder(DashboardNames.FUSION_BOARD, DashboardNames.ALGO_BOARD);
        }

        @Test
        @DisplayName("Should return empty list when current user has no MY_DASHBOARD")
        void shouldReturnEmptyListWhenUserHasNoDashboards() {
            Users newUser = createAndSaveUser("newUser");
            createAndSave(DashboardNames.FUSION_BOARD, "Other's dashboard", currentUserEntity, false);

            Specification<Dashboard> spec = DashboardSpecification.build(
                null,
                List.of(DashboardType.MY_DASHBOARD),
                newUser.getName(),
                false
            );

            List<Dashboard> results = dashboardRepository.findAll(spec);

            assertThat(results).isEmpty();
        }

        @Test
        @DisplayName("Should handle null search term")
        void shouldHandleNullSearchTerm() {
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);

            List<String> results = getDashboardNames(null, null, false);

            assertThat(results).isNotEmpty();
        }

        @Test
        @DisplayName("Should handle whitespace-only search term")
        void shouldHandleWhitespaceOnlySearchTerm() {
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);

            List<String> results = getDashboardNames("   ", null, false);

            // Whitespace should be treated as no search filter
            assertThat(results).isNotEmpty();
        }

        @Test
        @DisplayName("Should handle SQL special characters safely")
        void shouldHandleSqlSpecialCharacters() {
            createAndSave(DashboardNames.FUSION_BOARD, "My trading view", currentUserEntity, false);

            // Should not throw exception or cause SQL errors
            List<String> results = getDashboardNames("'; DROP TABLE dashboard; --", null, false);

            assertThat(results).isEmpty();
            // Verify table still exists
            assertThat(dashboardRepository.count()).isGreaterThan(0);
        }

        @Test
        @DisplayName("Should handle LIKE wildcards in search term")
        void shouldHandleLikeWildcardsInSearchTerm() {
            createAndSave("Dashboard 1", "First dashboard", currentUserEntity, false);
            createAndSave("Dashboard 2", "Second dashboard", currentUserEntity, false);

            // % should be treated as literal character, not wildcard
            List<String> results = getDashboardNames("%", null, false);

            assertThat(results).isEmpty();
        }
    }

    // ==================== Helper Methods ====================

    private Users createAndSaveUser(String name) {
        Users user = new Users();
        user.setName(name);
        return userRepository.save(user);
    }

    private Dashboard createDashboard(String name, String desc, Users user, boolean deleted) {
        Dashboard d = new Dashboard();
        d.setName(name);
        d.setDescription(desc);
        d.setUser(user);
        d.setDeleted(deleted);
        return d;
    }

    private Dashboard createAndSave(String name, String desc, Users user, boolean deleted) {
        Dashboard dashboard = createDashboard(name, desc, user, deleted);
        return dashboardRepository.save(dashboard);
    }

    private List<String> getDashboardNames(String search, List<DashboardType> types, boolean includeDeleted) {
        Specification<Dashboard> spec = DashboardSpecification.build(
            search,
            types,
            CURRENT_USER,
            includeDeleted
        );
        return dashboardRepository.findAll(spec)
            .stream()
            .map(Dashboard::getName)
            .toList();
    }
}
```
