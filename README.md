```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.jdbc.core.RowMapper;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Timestamp;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * Mapper class for TestTagStatisticsData with query building utilities.
 */
public class TestTagStatisticsMapper implements RowMapper<TestTagStatisticsData> {
    
    private static final Logger log = LoggerFactory.getLogger(TestTagStatisticsMapper.class);
    
    @Override
    public TestTagStatisticsData mapRow(final ResultSet rs, final int rowNum) throws SQLException {
        try {
            TestTagStatisticsData data = new TestTagStatisticsData();
            
            // Map basic fields
            data.setClassName(rs.getString("class_name"));
            data.setTagName(rs.getString("tag_name"));
            data.setPeriodIdx(rs.getInt("period_idx"));
            
            // Map timestamp fields
            Timestamp periodStart = rs.getTimestamp("period_start");
            if (periodStart != null) {
                data.setPeriodStart(periodStart.toLocalDateTime());
            }
            
            Timestamp periodEnd = rs.getTimestamp("period_end");
            if (periodEnd != null) {
                data.setPeriodEnd(periodEnd.toLocalDateTime());
            }
            
            // Map count fields
            data.setTotalTests(rs.getInt("total_tests"));
            data.setTestsWithRunInPeriod(rs.getInt("tests_with_run_in_period"));
            data.setPassedTests(rs.getInt("passed_tests"));
            
            // Map latest run time
            Timestamp latestRunTime = rs.getTimestamp("latest_run_time_for_cell");
            if (latestRunTime != null) {
                data.setLatestRunTime(latestRunTime.toLocalDateTime());
            }
            
            return data;
            
        } catch (SQLException e) {
            throw new SQLException("Error mapping TestTagStatisticsData at row " + rowNum, e);
        }
    }
    
    /**
     * Builds the complete query with the IN clause placeholders.
     *
     * @param query The base query template with %s placeholder for the IN clause.
     * @param tagCount The number of tags for the IN clause.
     * @return The complete query string with proper placeholders.
     */
    public static String buildQuery(final String query, final int tagCount) {
        if (tagCount <= 0) {
            log.error("Cannot build an IN clause with tagCount <= 0. Query: {}", query);
            throw new IllegalArgumentException(
                "tagCount must be greater than 0 for building an IN clause.");
        }
        
        final var placeholders = String.join(", ", Collections.nCopies(tagCount, "?"));
        return String.format(query, placeholders);
    }
    
    /**
     * Prepares an array of parameters for a JDBC query.
     *
     * @param startDate The start date string (e.g., "YYYY-MM-DD HH24:MI:SS").
     * @param endDate The end date string (e.g., "YYYY-MM-DD HH24:MI:SS").
     * @param appId The application ID.
     * @param regression A boolean indicating whether to filter by regression (true maps to 1, false to 0).
     * @param tagIds An array of tag IDs for the IN clause. Can be empty but not null if an IN clause is present.
     * @return An {@code Object[]} containing all parameters in the correct order for JDBC execution.
     */
    public static Object[] prepareParameters(
        final String startDate,
        final String endDate,
        final long appId,
        final boolean regression,
        final Long... tagIds) {
        
        final List<Object> params = new ArrayList<>();
        final var regressionValue = regression ? 1 : 0;
        
        // Parameters for the params CTE (3 parameters)
        // First occurrence: from_ts
        params.add(startDate);
        // Second occurrence: to_ts
        params.add(endDate);
        // Third occurrence: app_id
        params.add(appId);
        
        // Parameters for the tag IN clause
        // Add each tag ID to the parameters list
        for (Long tagId : tagIds) {
            params.add(tagId);
        }
        
        // Parameter for the regression filter
        params.add(regressionValue);
        
        return params.toArray();
    }
}

// ============================================================================
// DTO CLASS
// ============================================================================

/**
 * Data Transfer Object for test tag statistics.
 * Contains aggregated statistics for test runs grouped by class, tag, and time period.
 */
class TestTagStatisticsData {
    
    private String className;
    private String tagName;
    private Integer periodIdx;
    private LocalDateTime periodStart;
    private LocalDateTime periodEnd;
    private Integer totalTests;
    private Integer testsWithRunInPeriod;
    private Integer passedTests;
    private LocalDateTime latestRunTime;
    
    // Constructors
    public TestTagStatisticsData() {
    }
    
    public TestTagStatisticsData(String className, String tagName, Integer periodIdx,
                                 LocalDateTime periodStart, LocalDateTime periodEnd,
                                 Integer totalTests, Integer testsWithRunInPeriod,
                                 Integer passedTests, LocalDateTime latestRunTime) {
        this.className = className;
        this.tagName = tagName;
        this.periodIdx = periodIdx;
        this.periodStart = periodStart;
        this.periodEnd = periodEnd;
        this.totalTests = totalTests;
        this.testsWithRunInPeriod = testsWithRunInPeriod;
        this.passedTests = passedTests;
        this.latestRunTime = latestRunTime;
    }
    
    // Getters and Setters
    public String getClassName() {
        return className;
    }
    
    public void setClassName(String className) {
        this.className = className;
    }
    
    public String getTagName() {
        return tagName;
    }
    
    public void setTagName(String tagName) {
        this.tagName = tagName;
    }
    
    public Integer getPeriodIdx() {
        return periodIdx;
    }
    
    public void setPeriodIdx(Integer periodIdx) {
        this.periodIdx = periodIdx;
    }
    
    public LocalDateTime getPeriodStart() {
        return periodStart;
    }
    
    public void setPeriodStart(LocalDateTime periodStart) {
        this.periodStart = periodStart;
    }
    
    public LocalDateTime getPeriodEnd() {
        return periodEnd;
    }
    
    public void setPeriodEnd(LocalDateTime periodEnd) {
        this.periodEnd = periodEnd;
    }
    
    public Integer getTotalTests() {
        return totalTests;
    }
    
    public void setTotalTests(Integer totalTests) {
        this.totalTests = totalTests;
    }
    
    public Integer getTestsWithRunInPeriod() {
        return testsWithRunInPeriod;
    }
    
    public void setTestsWithRunInPeriod(Integer testsWithRunInPeriod) {
        this.testsWithRunInPeriod = testsWithRunInPeriod;
    }
    
    public Integer getPassedTests() {
        return passedTests;
    }
    
    public void setPassedTests(Integer passedTests) {
        this.passedTests = passedTests;
    }
    
    public LocalDateTime getLatestRunTime() {
        return latestRunTime;
    }
    
    public void setLatestRunTime(LocalDateTime latestRunTime) {
        this.latestRunTime = latestRunTime;
    }
    
    // Utility methods
    /**
     * Calculates the pass rate as a percentage.
     * @return Pass rate (0-100) or null if no tests ran in the period
     */
    public Double getPassRate() {
        if (testsWithRunInPeriod == null || testsWithRunInPeriod == 0) {
            return null;
        }
        return (passedTests != null ? passedTests : 0) * 100.0 / testsWithRunInPeriod;
    }
    
    /**
     * Calculates the coverage rate (tests that ran vs total tests).
     * @return Coverage rate (0-100) or null if no total tests
     */
    public Double getCoverageRate() {
        if (totalTests == null || totalTests == 0) {
            return null;
        }
        return (testsWithRunInPeriod != null ? testsWithRunInPeriod : 0) * 100.0 / totalTests;
    }
    
    @Override
    public String toString() {
        return "TestTagStatisticsData{" +
                "className='" + className + '\'' +
                ", tagName='" + tagName + '\'' +
                ", periodIdx=" + periodIdx +
                ", periodStart=" + periodStart +
                ", periodEnd=" + periodEnd +
                ", totalTests=" + totalTests +
                ", testsWithRunInPeriod=" + testsWithRunInPeriod +
                ", passedTests=" + passedTests +
                ", latestRunTime=" + latestRunTime +
                '}';
    }
}
```
