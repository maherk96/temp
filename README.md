```java

@Repository
public interface TestTagRepository extends JpaRepository<TestTag, Long> {

    @Query("""
        SELECT DISTINCT tt.tag
        FROM TestTag tt
        JOIN tt.testLaunch tl
        WHERE tl.app.id = :appId
          AND tl.regression = 1
          AND tl.created >= :threeMonthsAgo
          AND tt.tag IS NOT NULL
        ORDER BY tt.tag
        """)
    List<String> findDistinctTagsUsedInRegressionLast3MonthsByAppId(
        @Param("appId") Long appId,
        @Param("threeMonthsAgo") Instant threeMonthsAgo
    );
}

```
