```java
package com.example.service;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;
import org.springframework.context.annotation.Import;
import org.springframework.test.context.TestPropertySource;
import static org.junit.jupiter.api.Assertions.*;

import java.util.List;

@DataJpaTest
@Import({TestService.class, EntityHelper.class})
@TestPropertySource(properties = {
    "spring.jpa.hibernate.ddl-auto=create-drop",
    "spring.jpa.show-sql=true"
})
class TestServiceDatabaseIntegrationTest {

    @Autowired
    private TestEntityManager entityManager;

    @Autowired
    private TestRepository testRepository;

    @Autowired
    private TestService testService;

    private TestClass testClass;

    @BeforeEach
    void setUp() {
        // Create a test class for our tests
        testClass = new TestClass();
        testClass.setName("com.example.TestClass");
        testClass = entityManager.persistAndFlush(testClass);
    }

    @Test
    void shouldCreateSingleRecordForSameMethodNameAndDisplayName() {
        // Given - multiple calls with same method and display name (typical after extension fix)
        QAPTest test1 = new QAPTest("testMethod", "testMethod");
        QAPTest test2 = new QAPTest("testMethod", "testMethod");
        QAPTest test3 = new QAPTest("testMethod", "testMethod");

        // When - calling service multiple times
        long id1 = testService.getOrCreateTestDetails(test1, testClass.getId(), "TestClass");
        long id2 = testService.getOrCreateTestDetails(test2, testClass.getId(), "TestClass");
        long id3 = testService.getOrCreateTestDetails(test3, testClass.getId(), "TestClass");

        // Then - should return same ID and create only one record
        assertEquals(id1, id2);
        assertEquals(id1, id3);

        List<Test> allTests = testRepository.findByTestClassId(testClass.getId());
        assertEquals(1, allTests.size());
        
        Test savedTest = allTests.get(0);
        assertEquals("testMethod", savedTest.getMethodName());
        assertEquals("testMethod", savedTest.getDisplayName());
        assertEquals(id1, savedTest.getId());
    }

    @Test
    void shouldCreateSeparateRecordsForDifferentDisplayNames() {
        // Given - same method name but different display names
        QAPTest test1 = new QAPTest("testMethod", "Custom Display Name 1");
        QAPTest test2 = new QAPTest("testMethod", "Custom Display Name 2");

        // When
        long id1 = testService.getOrCreateTestDetails(test1, testClass.getId(), "TestClass");
        long id2 = testService.getOrCreateTestDetails(test2, testClass.getId(), "TestClass");

        // Then - should create separate records
        assertNotEquals(id1, id2);

        List<Test> allTests = testRepository.findByTestClassId(testClass.getId());
        assertEquals(2, allTests.size());

        // Verify both records exist with correct data
        Test test1Record = testRepository.findById(id1).orElseThrow();
        Test test2Record = testRepository.findById(id2).orElseThrow();

        assertEquals("testMethod", test1Record.getMethodName());
        assertEquals("Custom Display Name 1", test1Record.getDisplayName());
        
        assertEquals("testMethod", test2Record.getMethodName());
        assertEquals("Custom Display Name 2", test2Record.getDisplayName());
    }

    @Test
    void shouldNotCreateDuplicatesForParameterizedTestScenario() {
        // Given - simulating parameterized test runs with memory references (before extension fix)
        // and then with fixed display names (after extension fix)
        QAPTest beforeFix1 = new QAPTest("paramTest", "paramTest(TradeInfo@1a2b3c4d)");
        QAPTest beforeFix2 = new QAPTest("paramTest", "paramTest(TradeInfo@5e6f7g8h)");
        
        // These would have created duplicates before the fix
        long id1 = testService.getOrCreateTestDetails(beforeFix1, testClass.getId(), "TestClass");
        long id2 = testService.getOrCreateTestDetails(beforeFix2, testClass.getId(), "TestClass");

        // Now simulate after extension fix - both should map to same record
        QAPTest afterFix1 = new QAPTest("paramTest", "paramTest");
        QAPTest afterFix2 = new QAPTest("paramTest", "paramTest");
        
        long id3 = testService.getOrCreateTestDetails(afterFix1, testClass.getId(), "TestClass");
        long id4 = testService.getOrCreateTestDetails(afterFix2, testClass.getId(), "TestClass");

        // Then - before fix creates 2 records, after fix should reuse one of them
        assertNotEquals(id1, id2); // Different due to different display names
        assertEquals(id3, id4); // Same after fix
        
        // Should have 3 records total: 2 from before fix + 0 new (reuses existing)
        List<Test> allTests = testRepository.findByTestClassId(testClass.getId());
        assertEquals(2, allTests.size()); // Only the 2 from before fix

        // Verify one of the after-fix calls found an existing record
        assertTrue(id3 == id1 || id3 == id2);
    }

    @Test
    void shouldHandleExplicitDisplayNameConsistently() {
        // Given - test with explicit display name
        QAPTest test1 = new QAPTest("testMethod", "Meaningful Test Description");
        QAPTest test2 = new QAPTest("testMethod", "Meaningful Test Description");

        // When
        long id1 = testService.getOrCreateTestDetails(test1, testClass.getId(), "TestClass");
        long id2 = testService.getOrCreateTestDetails(test2, testClass.getId(), "TestClass");

        // Then - should reuse same record
        assertEquals(id1, id2);

        List<Test> allTests = testRepository.findByTestClassId(testClass.getId());
        assertEquals(1, allTests.size());

        Test savedTest = allTests.get(0);
        assertEquals("testMethod", savedTest.getMethodName());
        assertEquals("Meaningful Test Description", savedTest.getDisplayName());
    }

    @Test
    void shouldCreateSeparateRecordsForDifferentMethods() {
        // Given - different method names
        QAPTest test1 = new QAPTest("method1", "method1");
        QAPTest test2 = new QAPTest("method2", "method2");

        // When
        long id1 = testService.getOrCreateTestDetails(test1, testClass.getId(), "TestClass");
        long id2 = testService.getOrCreateTestDetails(test2, testClass.getId(), "TestClass");

        // Then
        assertNotEquals(id1, id2);

        List<Test> allTests = testRepository.findByTestClassId(testClass.getId());
        assertEquals(2, allTests.size());
    }

    @Test
    void shouldIsolateBetweenDifferentTestClasses() {
        // Given - another test class
        TestClass anotherTestClass = new TestClass();
        anotherTestClass.setName("com.example.AnotherTestClass");
        anotherTestClass = entityManager.persistAndFlush(anotherTestClass);

        QAPTest test1 = new QAPTest("sameMethod", "sameMethod");
        QAPTest test2 = new QAPTest("sameMethod", "sameMethod");

        // When - same method name in different classes
        long id1 = testService.getOrCreateTestDetails(test1, testClass.getId(), "TestClass");
        long id2 = testService.getOrCreateTestDetails(test2, anotherTestClass.getId(), "AnotherTestClass");

        // Then - should create separate records
        assertNotEquals(id1, id2);

        assertEquals(1, testRepository.findByTestClassId(testClass.getId()).size());
        assertEquals(1, testRepository.findByTestClassId(anotherTestClass.getId()).size());
    }

    @Test
    void shouldFindExistingRecordWithMethodNameOnlyStrategy() {
        // Given - create a record first
        Test existingTest = new Test();
        existingTest.setMethodName("existingMethod");
        existingTest.setDisplayName("existingMethod");
        existingTest.setTestClass(testClass);
        existingTest = entityManager.persistAndFlush(existingTest);

        QAPTest qapTest = new QAPTest("existingMethod", "existingMethod");

        // When
        long foundId = testService.getOrCreateTestDetails(qapTest, testClass.getId(), "TestClass");

        // Then - should find existing record, not create new one
        assertEquals(existingTest.getId(), foundId);

        List<Test> allTests = testRepository.findByTestClassId(testClass.getId());
        assertEquals(1, allTests.size()); // No new records created
    }

    @Test
    void shouldFindExistingRecordWithMethodAndDisplayNameStrategy() {
        // Given - create a record with custom display name
        Test existingTest = new Test();
        existingTest.setMethodName("existingMethod");
        existingTest.setDisplayName("Custom Display Name");
        existingTest.setTestClass(testClass);
        existingTest = entityManager.persistAndFlush(existingTest);

        QAPTest qapTest = new QAPTest("existingMethod", "Custom Display Name");

        // When
        long foundId = testService.getOrCreateTestDetails(qapTest, testClass.getId(), "TestClass");

        // Then
        assertEquals(existingTest.getId(), foundId);

        List<Test> allTests = testRepository.findByTestClassId(testClass.getId());
        assertEquals(1, allTests.size());
    }

    @Test
    void shouldPreventDuplicatesWithMixedDisplayNameFormats() {
        // Given - simulate the exact scenario from your original problem
        // Old record with auto-generated display name
        Test oldRecord = new Test();
        oldRecord.setMethodName("testPlaceMultipleTradesSuccessfully");
        oldRecord.setDisplayName("testPlaceMultipleTradesSuccessfully()");
        oldRecord.setTestClass(testClass);
        oldRecord = entityManager.persistAndFlush(oldRecord);

        // New calls with normalized display name (after extension fix)
        QAPTest newTest1 = new QAPTest("testPlaceMultipleTradesSuccessfully", "testPlaceMultipleTradesSuccessfully");
        QAPTest newTest2 = new QAPTest("testPlaceMultipleTradesSuccessfully", "testPlaceMultipleTradesSuccessfully");

        // When
        long id1 = testService.getOrCreateTestDetails(newTest1, testClass.getId(), "TestClass");
        long id2 = testService.getOrCreateTestDetails(newTest2, testClass.getId(), "TestClass");

        // Then - should create new record for normalized name, but reuse it for subsequent calls
        assertEquals(id1, id2); // Should reuse the same new record
        assertNotEquals(oldRecord.getId(), id1); // Should not find the old record due to different display name

        List<Test> allTests = testRepository.findByTestClassId(testClass.getId());
        assertEquals(2, allTests.size()); // Old record + new normalized record

        // Verify the new record has normalized display name
        Test newRecord = testRepository.findById(id1).orElseThrow();
        assertEquals("testPlaceMultipleTradesSuccessfully", newRecord.getMethodName());
        assertEquals("testPlaceMultipleTradesSuccessfully", newRecord.getDisplayName());
    }
}

```
