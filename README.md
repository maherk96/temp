```java
// Add these methods to your TestRepository interface

@Query("SELECT t FROM Test t WHERE t.testClass.id = ?1")
List<Test> findByTestClassId(Long testClassId);

@Query("SELECT t FROM Test t WHERE t.methodName = ?1 AND t.testClass.id = ?2")
List<Test> findByMethodNameAndTestClassId(String methodName, Long testClassId);

```
