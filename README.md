```java
/**
 * Specification for querying Dashboards based on search criteria and dashboard type.
 */
public class DashboardSpecification {

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

        // 🧩 Dashboard type filtering with proper LEFT JOIN
        if (dashboardTypes != null && !dashboardTypes.isEmpty()) {
            Specification<Dashboard> typeSpec = Specification.where(null);

            for (DashboardType type : dashboardTypes) {
                Specification<Dashboard> subSpec = (root, query, cb) -> {
                    Join<Dashboard, Users> userJoin = root.join("user", JoinType.LEFT);

                    return switch (type) {
                        case MY_DASHBOARD ->
                                cb.equal(userJoin.get("name"), currentUserId);
                        case CREATED_BY_OTHERS ->
                                cb.and(cb.isNotNull(userJoin.get("name")),
                                        cb.notEqual(userJoin.get("name"), currentUserId));
                        case SYSTEM_DASHBOARD ->
                                cb.isNull(root.get("user"));
                    };
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
