```java

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.LocalDateTime;
import java.util.*;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class HeatmapDataServiceTest {

    @Mock
    private DatabaseQueryExecutor queryExecutor;

    @Mock
    private HeatmapTagManagementService heatmapTagManagementService;

    @Mock
    private HeatmapManagementService heatmapManagementService;

    @Mock
    private TestTagStatisticsMapper mapper;

    @Captor
    private ArgumentCaptor<String> queryCaptor;

    @Captor
    private ArgumentCaptor<Object[]> paramsCaptor;

    private Map<String, String> qapQueries;

    private HeatmapDataService service;

    @BeforeEach
    void setup() {
        qapQueries = new HashMap<>();
        qapQueries.put("HeatmapAnalysisDataV2", "SELECT * FROM test WHERE tag IN (?)");

        // Use SPY so we can stub private-like methods (buildQuery, prepareParameters etc.)
        service = Mockito.spy(new HeatmapDataService(
                queryExecutor,
                qapQueries,
                heatmapTagManagementService,
                heatmapManagementService,
                mapper
        ));
    }

    // --------------------------------------------------------------------------------------
    // 1. Happy-path test for getHeatmapById()
    // --------------------------------------------------------------------------------------
    @Test
    void getHeatmapById_success() {
        long heatmapId = 10L;

        LocalDateTime start = LocalDateTime.now().minusDays(10);
        LocalDateTime end = LocalDateTime.now();

        // Mock: heatmap tags
        when(heatmapTagManagementService.getTagIdsByHeatmapId(heatmapId))
                .thenReturn(List.of(100L, 200L));

        // Mock: Heatmap entity
        Heatmap hm = mock(Heatmap.class);
        User user = mock(User.class);
        App app = mock(App.class);

        when(hm.getId()).thenReturn(10L);
        when(hm.getUser()).thenReturn(user);
        when(user.getName()).thenReturn("TestUser");
        when(hm.getName()).thenReturn("Sample Heatmap");
        when(hm.getDescription()).thenReturn("desc");
        when(hm.getDeleted()).thenReturn(false);
        when(hm.getApp()).thenReturn(app);
        when(app.getId()).thenReturn(50L);

        when(heatmapManagementService.findEntityById(heatmapId)).thenReturn(hm);

        // Mock: SQL builder + parameters
        doReturn("SQL_QUERY").when(service).buildQuery(anyString(), anyInt());
        doReturn(new Object[]{"param"}).when(service)
                .prepareParameters(any(), any(), any(), anyBoolean(), any());

        // Mock: status + period
        doReturn(HeatmapStatus.OK).when(service).determineStatus(any(), any());
        doReturn("Good").when(service).buildPeriodAnalysis(any(), any());

        // Mock: DB results
        TestTagStatisticsData row = mock(TestTagStatisticsData.class);
        when(row.getClassName()).thenReturn("Clazz");
        when(row.getTagName()).thenReturn("TagA");
        when(row.getLatestRunTime()).thenReturn(LocalDateTime.now());
        when(row.getPassedTests()).thenReturn(5);
        when(row.getTotalTests()).thenReturn(6);
        when(row.getPassRate()).thenReturn(80.0);

        when(queryExecutor.executeQuery(anyString(), eq(mapper), any()))
                .thenReturn(List.of(row));

        // Run
        HeatmapDetailsDTO dto =
                service.getHeatmapById(heatmapId, start, end, false);

        // Validate header
        assertEquals(10L, dto.header().id());
        assertEquals("TestUser", dto.header().createdBy());
        assertEquals("Sample Heatmap", dto.header().name());

        // Validate analysis section
        assertEquals(1, dto.data().size());
        HeatmapAnalysisDTO analysis = dto.data().get(0);

        assertEquals("Clazz", analysis.className());
        assertEquals("TagA", analysis.tagName());
        assertEquals(5, analysis.passedTests());
        assertEquals(80.0, analysis.passRate());
    }

    // --------------------------------------------------------------------------------------
    // 2. getHeatmapById(): ensure custom exception is thrown when service fails
    // --------------------------------------------------------------------------------------
    @Test
    void getHeatmapById_throwsHeatmapProcessingException() {
        when(heatmapTagManagementService.getTagIdsByHeatmapId(anyLong()))
                .thenThrow(new RuntimeException("DB fail"));

        assertThrows(
                HeatmapDataService.HeatmapProcessingException.class,
                () -> service.getHeatmapById(99L, LocalDateTime.now(), LocalDateTime.now(), false)
        );
    }

    // --------------------------------------------------------------------------------------
    // 3. getHeatmapAnalysisData(): ensure correct query + params passed to the DB executor
    // --------------------------------------------------------------------------------------
    @Test
    void getHeatmapAnalysisData_executesSqlCorrectly() {

        doReturn("FINAL_SQL").when(service).buildQuery(anyString(), anyInt());
        doReturn(new Object[]{"A"}).when(service)
                .prepareParameters(any(), any(), any(), anyBoolean(), any());

        when(queryExecutor.executeQuery(any(), eq(mapper), any()))
                .thenReturn(Collections.emptyList());

        service.getHeatmapAnalysisData(
                5L, LocalDateTime.now(), LocalDateTime.now(), new Long[]{1L, 2L}, false);

        verify(queryExecutor).executeQuery(
                queryCaptor.capture(),
                eq(mapper),
                paramsCaptor.capture()
        );

        assertEquals("FINAL_SQL", queryCaptor.getValue());
        assertArrayEquals(new Object[]{"A"}, paramsCaptor.getValue());
    }

    // --------------------------------------------------------------------------------------
    // 4. Most-recent-per-cell grouping logic
    // --------------------------------------------------------------------------------------
    @Test
    void buildHeatmapResponse_keepsMostRecentEntryOnly() throws Exception {

        // Create rows
        TestTagStatisticsData older = mock(TestTagStatisticsData.class);
        TestTagStatisticsData newer = mock(TestTagStatisticsData.class);

        when(older.getClassName()).thenReturn("ClassA");
        when(older.getTagName()).thenReturn("Tag1");
        when(older.getLatestRunTime()).thenReturn(LocalDateTime.now().minusDays(1));
        when(older.getPassedTests()).thenReturn(1);
        when(older.getTotalTests()).thenReturn(2);
        when(older.getPassRate()).thenReturn(50.0);

        when(newer.getClassName()).thenReturn("ClassA");
        when(newer.getTagName()).thenReturn("Tag1");
        when(newer.getLatestRunTime()).thenReturn(LocalDateTime.now());
        when(newer.getPassedTests()).thenReturn(5);
        when(newer.getTotalTests()).thenReturn(6);
        when(newer.getPassRate()).thenReturn(80.0);

        // Stub helpers
        doReturn(HeatmapStatus.OK).when(service).determineStatus(any(), any());
        doReturn("period").when(service).buildPeriodAnalysis(any(), any());

        // Call private-like method via spy (reflection not needed)
        List<HeatmapAnalysisDTO> results =
                service.buildHeatmapResponse(
                        List.of(older, newer),
                        LocalDateTime.now().minusDays(10)
                );

        assertEquals(1, results.size());
        assertEquals(80.0, results.get(0).passRate());  // only "newer" survives
    }
}

```
