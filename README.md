```java

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.*;
import java.util.stream.Collectors;

/**
 * Service responsible for retrieving Heatmap data, executing analysis queries,
 * aggregating results, and constructing DTOs for API responses.
 *
 * <p>This class orchestrates:</p>
 * <ul>
 *     <li>Loading heatmap metadata (app, user, tags)</li>
 *     <li>Executing the main SQL analysis query</li>
 *     <li>Selecting the most recent results per class/tag combination</li>
 *     <li>Mapping data to {@link HeatmapDetailsDTO} and child DTOs</li>
 * </ul>
 *
 * <p>The logic is intentionally separated into small, focused methods to
 * improve maintainability and simplify unit testing.</p>
 */
@Slf4j
@Service
public class HeatmapDataService {

    private static final String HEATMAP_ANALYSIS_QUERY_KEY = "HeatmapAnalysisDataV2";
    private static final DateTimeFormatter DATE_FMT =
            DateTimeFormatter.ofPattern("dd MMM yyyy");

    private final DatabaseQueryExecutor queryExecutor;
    private final Map<String, String> qapQueries;
    private final HeatmapTagManagementService heatmapTagManagementService;
    private final HeatmapManagementService heatmapManagementService;
    private final TestTagStatisticsMapper testTagStatisticsMapper;

    /**
     * Constructs the HeatmapDataService.
     *
     * @param queryExecutor                 Executes database SQL queries.
     * @param qapQueries                    A map of query names → SQL strings.
     * @param heatmapTagManagementService   Service for retrieving heatmap tag associations.
     * @param heatmapManagementService      Service for retrieving Heatmap entities.
     * @param testTagStatisticsMapper       Row mapper for query results.
     */
    @Autowired
    public HeatmapDataService(DatabaseQueryExecutor queryExecutor,
                              Map<String, String> qapQueries,
                              HeatmapTagManagementService heatmapTagManagementService,
                              HeatmapManagementService heatmapManagementService,
                              TestTagStatisticsMapper testTagStatisticsMapper) {

        this.queryExecutor = Objects.requireNonNull(queryExecutor, "queryExecutor");
        this.qapQueries = Objects.requireNonNull(qapQueries, "qapQueries");
        this.heatmapTagManagementService =
                Objects.requireNonNull(heatmapTagManagementService, "heatmapTagManagementService");
        this.heatmapManagementService =
                Objects.requireNonNull(heatmapManagementService, "heatmapManagementService");
        this.testTagStatisticsMapper =
                Objects.requireNonNull(testTagStatisticsMapper, "testTagStatisticsMapper");
    }

    /**
     * Executes the underlying SQL to load raw test/tag statistics for a given application
     * and tag selection within a date range.
     *
     * @param appId      ID of the application to filter by.
     * @param startDate  Start of the date range.
     * @param endDate    End of the date range.
     * @param tagIds     Tag IDs to include in the analysis.
     * @param regression If true, filters for regression results.
     * @return List of raw {@link TestTagStatisticsData} rows.
     * @throws IllegalStateException If the backing SQL query is missing.
     */
    public List<TestTagStatisticsData> getHeatmapAnalysisData(Long appId,
                                                              LocalDateTime startDate,
                                                              LocalDateTime endDate,
                                                              Long[] tagIds,
                                                              boolean regression) {

        String baseQuery = qapQueries.get(HEATMAP_ANALYSIS_QUERY_KEY);
        if (baseQuery == null) {
            throw new IllegalStateException(
                    "Missing query for key: " + HEATMAP_ANALYSIS_QUERY_KEY);
        }

        String query = buildQuery(baseQuery, tagIds.length);
        Object[] params = prepareParameters(
                toDbTimestamp(startDate),
                toDbTimestamp(endDate),
                appId,
                regression,
                tagIds
        );

        return queryExecutor.executeQuery(query, testTagStatisticsMapper, params);
    }

    /**
     * Loads all data required to build a complete {@link HeatmapDetailsDTO} response.
     * <p>This includes:</p>
     * <ul>
     *     <li>Heatmap metadata (name, description, user, tags)</li>
     *     <li>Most recent per-class/tag statistics</li>
     *     <li>Status and period analysis</li>
     * </ul>
     *
     * @param id         Heatmap ID.
     * @param startDate  Start of the reporting period.
     * @param endDate    End of the reporting period.
     * @param regression Whether to filter for regression results.
     * @return Full Heatmap details including aggregated analysis.
     * @throws HeatmapProcessingException If any processing error occurs.
     */
    public HeatmapDetailsDTO getHeatmapById(long id,
                                            LocalDateTime startDate,
                                            LocalDateTime endDate,
                                            boolean regression) {

        try {
            List<Long> tagIds =
                    heatmapTagManagementService.getTagIdsByHeatmapId(id);
            Heatmap heatmap =
                    heatmapManagementService.findEntityById(id);
            Long appId = heatmap.getApp().getId();

            List<TestTagStatisticsData> testTagData = getHeatmapAnalysisData(
                    appId, startDate, endDate, tagIds.toArray(new Long[0]), regression
            );

            HeatmapResponseDTO header = buildHeaderDto(heatmap, tagIds);
            List<HeatmapAnalysisDTO> analysis =
                    buildHeatmapResponse(testTagData, startDate);

            return new HeatmapDetailsDTO(header, analysis);

        } catch (Exception e) {
            log.error("Error generating weekly tag regression for heatmap {}: {}",
                    id, e.getMessage(), e);
            throw new HeatmapProcessingException(
                    "Failed to generate heatmap data for id " + id, e);
        }
    }

    /**
     * Builds the header DTO containing metadata about the heatmap.
     *
     * @param heatmap Heatmap entity.
     * @param tagIds  IDs of tags associated with this heatmap.
     * @return Constructed {@link HeatmapResponseDTO}.
     */
    private HeatmapResponseDTO buildHeaderDto(Heatmap heatmap, List<Long> tagIds) {
        String username = Optional.ofNullable(heatmap.getUser())
                .map(User::getName)
                .orElse("Unknown User");

        return new HeatmapResponseDTO(
                heatmap.getId(),
                username,
                heatmap.getName(),
                heatmap.getDescription(),
                heatmap.getDeleted(),
                heatmap.getApp().getId(),
                tagIds
        );
    }

    /**
     * Aggregates the raw data into a list of analysis DTOs by selecting the most recent
     * run per (className, tagName) pair and calculating status + period analysis.
     *
     * @param data          Raw statistics query results.
     * @param userStartDate Start of the user's reporting period.
     * @return Aggregated response ready for UI consumption.
     */
    private List<HeatmapAnalysisDTO> buildHeatmapResponse(List<TestTagStatisticsData> data,
                                                          LocalDateTime userStartDate) {

        if (data == null || data.isEmpty()) {
            return Collections.emptyList();
        }

        // Select most recent entry per class/tag key
        Collection<TestTagStatisticsData> mostRecentPerCell =
                data.stream()
                        .collect(Collectors.collectingAndThen(
                                Collectors.groupingBy(
                                        row -> row.getClassName() + "|" + row.getTagName(),
                                        Collectors.maxBy(Comparator.comparing(
                                                TestTagStatisticsData::getLatestRunTime))
                                ),
                                map -> map.values()
                                        .stream()
                                        .flatMap(Optional::stream)
                                        .toList()
                        ));

        return mostRecentPerCell.stream()
                .map(row -> toAnalysisDto(row, userStartDate))
                .toList();
    }

    /**
     * Converts a raw statistics row into a UI-ready {@link HeatmapAnalysisDTO}.
     *
     * @param row            Raw data row.
     * @param userStartDate  Reporting period start date.
     * @return Populated analysis DTO.
     */
    private HeatmapAnalysisDTO toAnalysisDto(TestTagStatisticsData row,
                                             LocalDateTime userStartDate) {

        LocalDateTime lastRun = row.getLatestRunTime();
        HeatmapStatus status = determineStatus(lastRun, userStartDate);
        String lastRunStr = lastRun != null
                ? lastRun.toLocalDate().format(DATE_FMT)
                : "N/A";

        String periodAnalysis = buildPeriodAnalysis(lastRun, userStartDate);

        return new HeatmapAnalysisDTO(
                row.getClassName(),
                row.getTagName(),
                row.getPassedTests(),
                row.getPassRate(),
                row.getTotalTests(),
                status,
                lastRunStr,
                periodAnalysis
        );
    }

    // ----------------------------------------------------------------------------------
    // Helper methods — JavaDoc included so the behaviour is clear
    // ----------------------------------------------------------------------------------

    /**
     * Builds a final SQL query based on the number of tag IDs.
     * Typically used to expand an IN-clause placeholder list.
     *
     * @param baseQuery Base SQL template.
     * @param tagCount  Number of tag IDs.
     * @return SQL string ready for parameter binding.
     */
    private String buildQuery(String baseQuery, int tagCount) {
        throw new UnsupportedOperationException("Implement buildQuery()");
    }

    /**
     * Prepares positional parameters for the heatmap SQL query.
     *
     * @param startTs Start timestamp.
     * @param endTs   End timestamp.
     * @param appId   Application ID.
     * @param regression Regression flag.
     * @param tagIds Tag IDs for filtering.
     * @return Array of parameters in correct order.
     */
    private Object[] prepareParameters(Object startTs,
                                       Object endTs,
                                       Long appId,
                                       boolean regression,
                                       Long[] tagIds) {
        throw new UnsupportedOperationException("Implement prepareParameters()");
    }

    /**
     * Converts a LocalDateTime to a DB-friendly timestamp object (e.g. java.sql.Timestamp).
     *
     * @param time Local date-time.
     * @return Database timestamp.
     */
    private Object toDbTimestamp(LocalDateTime time) {
        throw new UnsupportedOperationException("Implement toDbTimestamp()");
    }

    /**
     * Determines the heatmap status (OK, WARNING, FAIL, etc.).
     *
     * @param lastRun       Most recent run date.
     * @param userStartDate Reporting baseline start date.
     * @return Computed status.
     */
    private HeatmapStatus determineStatus(LocalDateTime lastRun,
                                          LocalDateTime userStartDate) {
        throw new UnsupportedOperationException("Implement determineStatus()");
    }

    /**
     * Builds a textual period analysis, e.g. “5 days since last run”.
     *
     * @param lastRun       Last execution date.
     * @param userStartDate Reporting baseline start date.
     * @return Human-readable analysis string.
     */
    private String buildPeriodAnalysis(LocalDateTime lastRun,
                                       LocalDateTime userStartDate) {
        throw new UnsupportedOperationException("Implement buildPeriodAnalysis()");
    }

    // ----------------------------------------------------------------------------------

    /**
     * Thrown when a heatmap cannot be processed due to unexpected errors.
     */
    public static class HeatmapProcessingException extends RuntimeException {
        public HeatmapProcessingException(String message, Throwable cause) {
            super(message, cause);
        }
    }
}

```
