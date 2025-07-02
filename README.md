```java
@Query("""
  SELECT tr FROM TestRun tr
  JOIN tr.test t
  JOIN t.testClass tc
  WHERE tr.testLaunch.id = :launchId
    AND tc.id = :classId
  ORDER BY t.methodName
""")
List<TestRun> findByLaunchIdAndClassId(
    @Param("launchId") Long launchId,
    @Param("classId") Long classId
);
```
