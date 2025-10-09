```java
/**
 * Builds a Specification for querying Dashboards.
 *
 * @param search the search term to filter by name or description
 * @param dashboardTypes the list of dashboard types to filter by (e.g., MY_DASHBOARD, CREATED_BY_OTHERS, SYSTEM_DASHBOARD)
 * @param currentUserId the ID of the current user for filtering
 * @param deleted whether to include deleted dashboards
 * @return a Specification for querying Dashboards
 */
public static Specification<Dashboard> build(
        String search,
        List<DashboardType> dashboardTypes,
        String currentUserId,
        boolean deleted) {

    Specification<Dashboard> spec = Specification.where(null);

    // 🔍 Search filter (name or description contains text)
    if (StringUtils.hasText(search)) {
        spec = spec.and((root, query, cb) -> cb.or(
                cb.like(cb.lower(root.get("name")), "%" + search.toLowerCase() + "%"),
                cb.like(cb.lower(root.get("description")), "%" + search.toLowerCase() + "%")
        ));
    }

    // 🧩 Dashboard type filters (supports multiple)
    if (dashboardTypes != null && !dashboardTypes.isEmpty()) {
        Specification<Dashboard> typeSpec = Specification.where(null);

        for (DashboardType type : dashboardTypes) {
            Specification<Dashboard> subSpec = switch (type) {
                case MY_DASHBOARD -> (root, query, cb) ->
                        cb.equal(root.get("userId"), Long.valueOf(currentUserId));

                case CREATED_BY_OTHERS -> (root, query, cb) ->
                        cb.and(
                                cb.isNotNull(root.get("userId")),
                                cb.notEqual(root.get("userId"), Long.valueOf(currentUserId))
                        );

                case SYSTEM_DASHBOARD -> (root, query, cb) ->
                        cb.isNull(root.get("userId"));
            };
            typeSpec = (typeSpec == null) ? subSpec : typeSpec.or(subSpec);
        }

        spec = spec.and(typeSpec);
    }

    // 🚫 Deleted filter (by default exclude deleted)
    if (!deleted) {
        spec = spec.and((root, query, cb) -> cb.isFalse(root.get("deleted")));
    }

    return spec;
}

```
