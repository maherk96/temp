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
        return (root, query, cb) -> {
            // ✅ create reusable LEFT JOIN once
            Join<Dashboard, Users> userJoin = root.join("user", JoinType.LEFT);

            var predicates = cb.conjunction();

            // 🔍 Search filter
            if (StringUtils.hasText(search)) {
                predicates.getExpressions().add(
                        cb.or(
                                cb.like(cb.lower(root.get("name")), "%" + search.toLowerCase() + "%"),
                                cb.like(cb.lower(root.get("description")), "%" + search.toLowerCase() + "%")
                        )
                );
            }

            // 🧩 Dashboard type filtering (with single join)
            if (dashboardTypes != null && !dashboardTypes.isEmpty()) {
                var typePredicates = cb.disjunction(); // OR logic between dashboard types

                for (DashboardType type : dashboardTypes) {
                    switch (type) {
                        case MY_DASHBOARD -> typePredicates.getExpressions().add(
                                cb.equal(userJoin.get("name"), currentUserId)
                        );
                        case CREATED_BY_OTHERS -> typePredicates.getExpressions().add(
                                cb.and(
                                        cb.isNotNull(userJoin.get("name")),
                                        cb.notEqual(userJoin.get("name"), currentUserId)
                                )
                        );
                        case SYSTEM_DASHBOARD -> typePredicates.getExpressions().add(
                                cb.isNull(root.get("user"))
                        );
                    }
                }

                predicates.getExpressions().add(typePredicates);
            }

            // 🧹 Deleted filter
            if (!deleted) {
                predicates.getExpressions().add(cb.isFalse(root.get("deleted")));
            }

            return predicates;
        };
    }
}
```
