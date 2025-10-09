```java

package com.citi.fx.qa.qap.db.repository;

import com.citi.fx.qa.qap.db.entity.Dashboard;
import com.citi.fx.qa.qap.db.entity.Users;
import com.citi.fx.qa.qap.db.enums.DashboardType;
import com.citi.fx.qa.qap.db.specification.DashboardSpecification;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.data.jpa.domain.Specification;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
class DashboardSpecificationIT {

    @Autowired
    private DashboardRepository dashboardRepository;

    private static final String CURRENT_USER = "mk66440";

    private Users currentUserEntity;
    private Users otherUserEntity;

    @BeforeEach
    void setup() {
        currentUserEntity = new Users();
        currentUserEntity.setName(CURRENT_USER);

        otherUserEntity = new Users();
        otherUserEntity.setName("otherUser");

        // Create sample dashboards
        Dashboard myDashboard = createDashboard("Fusion Board", "My trading view", currentUserEntity, false);
        Dashboard othersDashboard = createDashboard("Algo Board", "Other user's algo view", otherUserEntity, false);
        Dashboard systemDashboard = createDashboard("Regression Monitor", "System default", null, false);
        Dashboard deletedDashboard = createDashboard("Old Dashboard", "Soft deleted test", currentUserEntity, true);

        dashboardRepository.saveAll(List.of(myDashboard, othersDashboard, systemDashboard, deletedDashboard));
    }

    private Dashboard createDashboard(String name, String desc, Users user, boolean deleted) {
        Dashboard d = new Dashboard();
        d.setName(name);
        d.setDescription(desc);
        d.setUser(user);
        d.setDeleted(deleted);
        return d;
    }

    // 1️⃣ MY_DASHBOARD — Only dashboards owned by current user
    @Test
    void shouldReturnOnlyMyDashboards_whenTypeIsMyDashboard() {
        Specification<Dashboard> spec = DashboardSpecification.build(
                null,
                List.of(DashboardType.MY_DASHBOARD),
                CURRENT_USER,
                false);

        List<Dashboard> results = dashboardRepository.findAll(spec);

        assertThat(results)
                .extracting(Dashboard::getName)
                .containsExactlyInAnyOrder("Fusion Board")
                .doesNotContain("Algo Board", "Regression Monitor", "Old Dashboard");
    }

    // 2️⃣ CREATED_BY_OTHERS — Dashboards owned by someone else
    @Test
    void shouldReturnDashboardsCreatedByOthers_whenTypeIsCreatedByOthers() {
        Specification<Dashboard> spec = DashboardSpecification.build(
                null,
                List.of(DashboardType.CREATED_BY_OTHERS),
                CURRENT_USER,
                false);

        List<Dashboard> results = dashboardRepository.findAll(spec);

        assertThat(results)
                .extracting(Dashboard::getName)
                .containsExactlyInAnyOrder("Algo Board")
                .doesNotContain("Fusion Board", "Regression Monitor", "Old Dashboard");
    }

    // 3️⃣ SYSTEM_DASHBOARD — Dashboards with null user
    @Test
    void shouldReturnSystemDashboards_whenTypeIsSystemDashboard() {
        Specification<Dashboard> spec = DashboardSpecification.build(
                null,
                List.of(DashboardType.SYSTEM_DASHBOARD),
                CURRENT_USER,
                false);

        List<Dashboard> results = dashboardRepository.findAll(spec);

        assertThat(results)
                .extracting(Dashboard::getName)
                .containsExactlyInAnyOrder("Regression Monitor")
                .doesNotContain("Fusion Board", "Algo Board", "Old Dashboard");
    }

    // 4️⃣ MULTIPLE TYPES — My dashboards + system dashboards
    @Test
    void shouldReturnMyAndSystemDashboards_whenMultipleTypesAreGiven() {
        Specification<Dashboard> spec = DashboardSpecification.build(
                null,
                List.of(DashboardType.MY_DASHBOARD, DashboardType.SYSTEM_DASHBOARD),
                CURRENT_USER,
                false);

        List<Dashboard> results = dashboardRepository.findAll(spec);

        assertThat(results)
                .extracting(Dashboard::getName)
                .containsExactlyInAnyOrder("Fusion Board", "Regression Monitor")
                .doesNotContain("Algo Board", "Old Dashboard");
    }

    // 5️⃣ SEARCH — Search by name or description (case-insensitive)
    @Test
    void shouldReturnDashboardsMatchingSearchTerm() {
        Specification<Dashboard> spec = DashboardSpecification.build(
                "algo",
                null,
                CURRENT_USER,
                false);

        List<Dashboard> results = dashboardRepository.findAll(spec);

        assertThat(results)
                .extracting(Dashboard::getName)
                .containsExactlyInAnyOrder("Algo Board");
    }

    // 6️⃣ DELETED — Should include deleted dashboards if deleted=true
    @Test
    void shouldIncludeDeletedDashboards_whenDeletedTrue() {
        Specification<Dashboard> spec = DashboardSpecification.build(
                null,
                List.of(DashboardType.MY_DASHBOARD),
                CURRENT_USER,
                true);

        List<Dashboard> results = dashboardRepository.findAll(spec);

        assertThat(results)
                .extracting(Dashboard::getName)
                .contains("Fusion Board", "Old Dashboard")
                .doesNotContain("Algo Board", "Regression Monitor");
    }

    // 7️⃣ DEFAULT BEHAVIOR — Should exclude deleted dashboards if deleted=false
    @Test
    void shouldExcludeDeletedDashboards_whenDeletedFalse() {
        Specification<Dashboard> spec = DashboardSpecification.build(
                null,
                List.of(DashboardType.MY_DASHBOARD, DashboardType.SYSTEM_DASHBOARD),
                CURRENT_USER,
                false);

        List<Dashboard> results = dashboardRepository.findAll(spec);

        assertThat(results)
                .extracting(Dashboard::getName)
                .doesNotContain("Old Dashboard");
    }
}

```
