```java
public long getOrCreateTestDetails(QAPTest testDescription, long testClassId, String clazzName) {
    var methodName = testDescription.getMethodName();
    var displayName = testDescription.getDisplayName();
    
    // If they're the same, only check by methodName to avoid duplicates
    if (methodName.equals(displayName)) {
        return EntityHelper.getOrCreate(
            () -> methodExistsForTestClass(methodName, testClassId),
            () -> getTestCaseByMethodName(methodName, testClassId).getId(),
            () -> TestDTO.createTestDetails(displayName, methodName, testClassId),
            this::createForJUnit,
            String.format("Existing test case found for method %s in %s", methodName, clazzName),
            String.format("Adding test case %s to test class %s", methodName, clazzName)
        );
    }
    
    // Only for explicitly different display names
    return EntityHelper.getOrCreate(
        () -> checkIfTestCaseExistsInTestClass(methodName, displayName, testClassId),
        () -> getTestCaseForTestClass(methodName, displayName, testClassId).getId(),
        () -> TestDTO.createTestDetails(displayName, methodName, testClassId),
        this::createForJUnit,
        String.format("Existing test case found for %s %s %s", methodName, displayName, clazzName),
        String.format("Adding test case %s to test class %s %s", methodName, displayName, clazzName)
    );
}
@Query("select (count(t) > 0) from Test t where t.methodName = ?1 and t.testClass.id = ?2")
boolean methodExistsForTestClass(String methodName, Long testClassId);

@Query("select t from Test t where t.methodName = ?1 and t.testClass.id = ?2")
Test getTestCaseByMethodName(String methodName, Long testClassId);

```
