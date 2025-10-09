```java



/**
 * Specification for querying Dashboards based on search criteria and dashboard type.
 */
public class DashboardSpecification {

    /**
     * Builds a Specification for querying Dashboards.
     *
     * @param search          the search term to filter by name or description
     * @param dashboardTypes  the list of dashboard types to filter by
     *                        (e.g., MY_DASHBOARD, CREATED_BY_OTHERS, SYSTEM_DASHBOARD)
     * @param currentUserId   the ID (SOEID / username) of the current user
     * @param deleted         whether to include deleted dashboards
     * @return a Specification for querying Dashboards
     */
    public static Specification<Dashboard> build(
            String search,
            List<DashboardType> dashboardTypes,
            String currentUserId,
            boolean deleted
    ) {
        Specification<Dashboard> spec = Specification.where(null);

        // 🔍 Search by name or description (case-insensitive)
        if (StringUtils.hasText(search)) {
            spec = spec.and((root, query, cb) ->
                    cb.or(
                            cb.like(cb.lower(root.get("name")), "%" + search.toLowerCase() + "%"),
                            cb.like(cb.lower(root.get("description")), "%" + search.toLowerCase() + "%")
                    )
            );
        }

        // 🧩 Filter by dashboard types (MY, OTHERS, SYSTEM)
        if (dashboardTypes != null && !dashboardTypes.isEmpty()) {
            Specification<Dashboard> typeSpec = Specification.where(null);

            for (DashboardType type : dashboardTypes) {
                Specification<Dashboard> subSpec = switch (type) {
                    case MY_DASHBOARD ->
                            (root, query, cb) -> cb.equal(root.get("user").get("name"), currentUserId);
                    case CREATED_BY_OTHERS ->
                            (root, query, cb) -> cb.notEqual(root.get("user").get("name"), currentUserId);
                    case SYSTEM_DASHBOARD ->
                            (root, query, cb) -> cb.isNull(root.get("user"));
                };

                // Combine multiple dashboard types with OR logic
                typeSpec = (typeSpec == null) ? subSpec : typeSpec.or(subSpec);
            }

            spec = spec.and(typeSpec);
        }

        // 🧹 Exclude deleted dashboards unless explicitly requested
        if (!deleted) {
            spec = spec.and((root, query, cb) -> cb.isFalse(root.get("deleted")));
        }

        return spec;
    }
}
```
