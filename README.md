```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;

import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class QAPQueryServiceTest {

    @Mock
    private DatabaseQueryExecutor queryExecutor;

    @Mock
    private RowMapper<String> rowMapper;

    private QAPQueryService queryService;

    private final Map<String, String> queries = Map.of(
        "TestQuery", "SELECT * FROM test_table WHERE col = ?"
    );

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        queryService = new QAPQueryService(queryExecutor, queries);
    }

    @Test
    void shouldExecuteQueryAndReturnList() {
        List<String> expected = List.of("A", "B");
        when(queryExecutor.executeQuery(anyString(), any(), any())).thenReturn(expected);

        List<String> result = queryService.executeQuery("TestQuery", rowMapper, "someParam");

        assertEquals(expected, result);
        verify(queryExecutor).executeQuery(eq("SELECT * FROM test_table WHERE col = ?"), eq(rowMapper), eq("someParam"));
    }

    @Test
    void shouldThrowIfQueryNotFound_executeQuery() {
        Exception ex = assertThrows(IllegalArgumentException.class, () ->
            queryService.executeQuery("UnknownQuery", rowMapper));

        assertEquals("Query not found: UnknownQuery", ex.getMessage());
    }

    @Test
    void shouldRunSingleResultQueryAndReturnFirstItem() {
        when(queryExecutor.executeQuery(anyString(), any(), any()))
            .thenReturn(List.of("OnlyResult"));

        String result = queryService.runSingleResultQuery("TestQuery", rowMapper, "id");

        assertEquals("OnlyResult", result);
    }

    @Test
    void shouldThrowIfQueryNotFound_runSingleResultQuery() {
        Exception ex = assertThrows(IllegalArgumentException.class, () ->
            queryService.runSingleResultQuery("InvalidQuery", rowMapper));

        assertEquals("Query not found: InvalidQuery", ex.getMessage());
    }

    @Test
    void shouldThrowIfResultIsEmpty_runSingleResultQuery() {
        when(queryExecutor.executeQuery(anyString(), any(), any()))
            .thenReturn(List.of());

        // This will throw IndexOutOfBoundsException (from .get(0)), which is expected
        assertThrows(IndexOutOfBoundsException.class, () ->
            queryService.runSingleResultQuery("TestQuery", rowMapper));
    }
}


```
