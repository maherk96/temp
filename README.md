```java

    public static Specification<Dashboard> build(
            String search,
            List<DashboardType> dashboardTypes,
            Long currentUserId, // switched from SOEID string
            boolean includeDeleted) {

        return (root, query, cb) -> {
            List<Predicate> finalPredicates = new ArrayList<>();

            // 🔍 Search by name or description
            if (StringUtils.hasText(search)) {
                String q = "%" + search.toLowerCase() + "%";
                Predicate searchPredicate = cb.or(
                        cb.like(cb.lower(root.get("name")), q),
                        cb.like(cb.lower(root.get("description")), q)
                );
                finalPredicates.add(searchPredicate);
            }

            if (dashboardTypes != null && !dashboardTypes.isEmpty()) {
                List<Predicate> typeOrPredicates = new ArrayList<>();

                for (DashboardType t : dashboardTypes) {
                    Predicate subPredicate = switch (t) {
                        case MY_DASHBOARD -> cb.equal(root.get("user").get("id"), currentUserId);
                        case CREATED_BY_OTHERS -> cb.and(
                                cb.notEqual(root.get("user").get("id"), currentUserId),
                                cb.isFalse(root.get("isSystem"))
                        );
                        case SYSTEM_DASHBOARD -> cb.isTrue(root.get("isSystem"));
                        default -> null;
                    };

                    if (subPredicate != null) {
                        typeOrPredicates.add(subPredicate);
                    }
                }

                if (!typeOrPredicates.isEmpty()) {
                    finalPredicates.add(cb.or(typeOrPredicates.toArray(new Predicate[0])));
                }
            }

            if (!includeDeleted) {
                finalPredicates.add(cb.isFalse(root.get("deleted")));
            }

            return cb.and(finalPredicates.toArray(new Predicate[0]));
        };
    }
}
```
