```java
public enum DashboardType { MY_DASHBOARD, CREATED_BY_OTHERS, SYSTEM_DASHBOARD }

public final class DashboardSpecification {
    private static final String SYSTEM_SOEID = "system";

    public static Specification<Dashboard> build(
            String search,
            List<DashboardType> dashboardTypes,
            String currentUserSoeid,
            boolean includeDeleted
    ) {
        Specification<Dashboard> spec = Specification.where(null);

        // search (case-insensitive on name/description)
        if (org.springframework.util.StringUtils.hasText(search)) {
            String q = "%" + search.toLowerCase() + "%";
            spec = spec.and((root, query, cb) ->
                    cb.or(
                        cb.like(cb.lower(root.get("name")), q),
                        cb.like(cb.lower(root.get("description")), q)
                    )
            );
        }

        // type filtering (OR across types)
        if (dashboardTypes != null && !dashboardTypes.isEmpty()) {
            Specification<Dashboard> typeSpec = Specification.where(null);

            for (DashboardType t : dashboardTypes) {
                Specification<Dashboard> one = switch (t) {
                    case MY_DASHBOARD ->
                        (root, q, cb) -> cb.equal(root.get("user").get("name"), currentUserSoeid);

                    case CREATED_BY_OTHERS ->
                        (root, q, cb) -> cb.and(
                                cb.notEqual(root.get("user").get("name"), currentUserSoeid),
                                cb.notEqual(root.get("user").get("name"), SYSTEM_SOEID)
                        );

                    case SYSTEM_DASHBOARD ->
                        (root, q, cb) -> cb.equal(root.get("user").get("name"), SYSTEM_SOEID);
                };
                typeSpec = (typeSpec == null) ? one : typeSpec.or(one);
            }
            spec = spec.and(typeSpec);
        }

        if (!includeDeleted) {
            spec = spec.and((root, q, cb) -> cb.isFalse(root.get("deleted")));
        }
        return spec;
    }

    private DashboardSpecification() {}
}

```
