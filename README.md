```java

   /**
     * Retrieves all dashboards based on search, type, user, and pagination — with caching.
     */
    @Cacheable(
        cacheNames = "dashboardCache",
        key = "'findAll_' + #search + '_' + T(java.util.Objects).hash(#dashboardType) + '_' + #currentUserId + '_' + #deleted + '_' + #pageable.pageNumber + '_' + #pageable.pageSize"
    )
```
