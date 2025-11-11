```java
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Timestamp;
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;
import org.springframework.jdbc.core.RowMapper;

public class TestTagStatisticsMapper implements RowMapper<TestTagStatisticsData> {

    @Override
    public TestTagStatisticsData mapRow(ResultSet rs, int rowNum) throws SQLException {
        return new TestTagStatisticsData(
            rs.getString("testClassName"),
            rs.getString("tag"),
            rs.getLong("tagId"),
            rs.getInt("passed"),
            rs.getInt("totalTestRuns"),
            rs.getTimestamp("lastRun") != null ? rs.getTimestamp("lastRun").toString() : null,
            rs.getString("testRunIds")
        );
    }
    
    /**
     * Parameterized SQL query for test tag statistics.
     * 
     * @param startDate Start date for the time range (YYYY-MM-DD HH24:MI:SS format)
     * @param endDate End date for the time range (YYYY-MM-DD HH24:MI:SS format)
     * @param appId Application ID filter
     * @param tagIds List of tag IDs to filter by
     * @return Parameterized SQL query string
     */
    public static String getParameterizedQuery() {
        return """
            SELECT
                COALESCE(tc.NAME, 'N/A') AS testClassName,
                tg.NAME AS tag,
                tg.ID AS tagId,
                SUM(
                    CASE
                        WHEN tl.REGRESSION = 1
                         AND tl.CREATED BETWEEN TO_TIMESTAMP(?, 'YYYY-MM-DD HH24:MI:SS')
                                             AND TO_TIMESTAMP(?, 'YYYY-MM-DD HH24:MI:SS')
                         AND tr.STATUS != 'FAILED'
                        THEN 1
                        ELSE 0
                    END
                ) AS passed,
                COUNT(DISTINCT CASE
                    WHEN tl.REGRESSION = 1
                     AND tl.CREATED BETWEEN TO_TIMESTAMP(?, 'YYYY-MM-DD HH24:MI:SS')
                                         AND TO_TIMESTAMP(?, 'YYYY-MM-DD HH24:MI:SS')
                    THEN tr.ID
                    ELSE NULL
                END) AS totalTestRuns,
                COALESCE(
                    MAX(CASE
                        WHEN tl.REGRESSION = 1
                         AND tl.CREATED BETWEEN TO_TIMESTAMP(?, 'YYYY-MM-DD HH24:MI:SS')
                                             AND TO_TIMESTAMP(?, 'YYYY-MM-DD HH24:MI:SS')
                        THEN tr.START_TIME
                        ELSE NULL
                    END),
                    MAX(CASE
                        WHEN tr.START_TIME < TO_TIMESTAMP(?, 'YYYY-MM-DD HH24:MI:SS')
                        THEN tr.START_TIME
                        ELSE NULL
                    END)
                ) AS lastRun,
                LISTAGG(
                    CASE
                        WHEN tl.REGRESSION = 1
                         AND tl.CREATED BETWEEN TO_TIMESTAMP(?, 'YYYY-MM-DD HH24:MI:SS')
                                             AND TO_TIMESTAMP(?, 'YYYY-MM-DD HH24:MI:SS')
                        THEN tr.ID
                        ELSE NULL
                    END, ','
                ) WITHIN GROUP (ORDER BY tr.ID) AS testRunIds
            FROM
                QAPORTAL.TAG tg
            LEFT JOIN
                QAPORTAL.TEST_TAG tt ON tg.ID = tt.TAG_ID
            LEFT JOIN
                QAPORTAL.TEST t ON tt.TEST_ID = t.ID
            LEFT JOIN
                QAPORTAL.TEST_CLASS tc ON t.TEST_CLASS_ID = tc.ID
            LEFT JOIN
                QAPORTAL.TEST_LAUNCH tl ON tt.TEST_LAUNCH_ID = tl.ID AND tl.APP_ID = ?
            LEFT JOIN
                QAPORTAL.TEST_RUN tr ON tl.ID = tr.TEST_LAUNCH_ID AND t.ID = tr.TEST_ID
            WHERE
                tg.ID IN (%s)
            GROUP BY
                tc.NAME,
                tg.NAME,
                tg.ID
            ORDER BY
                testClassName,
                tag
            """;
    }
    
    /**
     * Builds the complete parameterized query with IN clause placeholders.
     * 
     * @param tagCount Number of tag IDs in the IN clause
     * @return Complete query with proper number of placeholders
     */
    public static String buildQuery(int tagCount) {
        String placeholders = String.join(",", 
            java.util.Collections.nCopies(tagCount, "?"));
        return String.format(getParameterizedQuery(), placeholders);
    }
    
    /**
     * Prepares parameters array for the query.
     * 
     * @param startDate Start date string (YYYY-MM-DD HH24:MI:SS)
     * @param endDate End date string (YYYY-MM-DD HH24:MI:SS)
     * @param appId Application ID
     * @param tagIds Array of tag IDs
     * @return Object array with all parameters in correct order
     */
    public static Object[] prepareParameters(String startDate, String endDate, 
                                            Integer appId, Long... tagIds) {
        // Parameters appear 9 times in the query + 1 for appId + tagIds.length for IN clause
        Object[] params = new Object[10 + tagIds.length];
        
        int idx = 0;
        // First BETWEEN (passed calculation)
        params[idx++] = startDate;
        params[idx++] = endDate;
        
        // Second BETWEEN (totalTestRuns calculation)
        params[idx++] = startDate;
        params[idx++] = endDate;
        
        // Third BETWEEN (lastRun MAX CASE)
        params[idx++] = startDate;
        params[idx++] = endDate;
        
        // Fourth comparison (lastRun fallback)
        params[idx++] = startDate;
        
        // Fifth BETWEEN (LISTAGG)
        params[idx++] = startDate;
        params[idx++] = endDate;
        
        // APP_ID filter
        params[idx++] = appId;
        
        // Tag IDs for IN clause
        for (Long tagId : tagIds) {
            params[idx++] = tagId;
        }
        
        return params;
    }
}

// Data record class
record TestTagStatisticsData(
    String testClassName,
    String tag,
    Long tagId,
    Integer passed,
    Integer totalTestRuns,
    String lastRun,
    String testRunIds
) {
    
    /**
     * Parses the comma-separated test run IDs into a list.
     * @return List of test run IDs, or empty list if null/empty
     */
    public List<Long> getTestRunIdsList() {
        if (testRunIds == null || testRunIds.isBlank()) {
            return List.of();
        }
        return Arrays.stream(testRunIds.split(","))
            .filter(s -> !s.isBlank() && !s.equals("null"))
            .map(String::trim)
            .map(Long::parseLong)
            .collect(Collectors.toList());
    }
}
```
