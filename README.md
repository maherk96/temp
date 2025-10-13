```java

public interface TestLaunchRepository extends JpaRepository<TestLaunch, Long> {

    Optional<TestLaunch> findByLaunchId(String launchId);

    TestLaunch findTopByApplication_NameAndEnvironment_NameOrderByStartTimeDesc(
        String appName, String envName);

    @Query("""
           SELECT t.launchId
           FROM TestLaunch t
           WHERE t.environment.name = :environment
           ORDER BY t.endTime DESC
           """)
    List<String> findLatestLaunchIdsByEnv(@Param("environment") String environment, Pageable pageable);

    default String findLatestLaunchIdByEnv(String environment) {
        return findLatestLaunchIdsByEnv(environment, PageRequest.of(0, 1))
                .stream()
                .findFirst()
                .orElse(null);
    }
}


```
