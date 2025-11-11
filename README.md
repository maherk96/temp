```java
@Override
    public TestTagStatisticsData mapRow(ResultSet rs, int rowNum) throws SQLException {
        // Handle potential null values from aggregation functions
        Long tagId = rs.getObject("tagId") != null ? rs.getLong("tagId") : null;
        Integer passed = rs.getObject("passed") != null ? rs.getInt("passed") : 0;
        Integer totalTestRuns = rs.getObject("totalTestRuns") != null ? rs.getInt("totalTestRuns") : 0;
        
        // Handle timestamp - could be null if no data in date range
        String lastRun = null;
        if (rs.getTimestamp("lastRun") != null) {
            lastRun = rs.getTimestamp("lastRun").toString();
        }
        
        // Handle LISTAGG result - could be null or contain "null" values
        String testRunIds = rs.getString("testRunIds");
        if (testRunIds != null) {
            // Clean up any "null" strings that might appear from LISTAGG
            testRunIds = testRunIds.replaceAll("(^|,)null(,|$)", "$1$2")
                                   .replaceAll("^,|,$", "")
                                   .replaceAll(",,+", ",");
            if (testRunIds.isBlank()) {
                testRunIds = null;
            }
        }
        
        return new TestTagStatisticsData(
            rs.getString("testClassName"),
            rs.getString("tag"),
            tagId,
            passed,
            totalTestRuns,
            lastRun,
            testRunIds
        );
    }

```
