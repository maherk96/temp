```sql

CREATE TABLE TEST_CLASS_SUMMARY (
    test_class_id NUMBER(19) NOT NULL,
    test_class_name VARCHAR2(100),
    tag VARCHAR2(50) NOT NULL,
    week_start_date DATE NOT NULL,
    week_end_date DATE NOT NULL,
    passed_count NUMBER(10) DEFAULT 0,
    total_count NUMBER(10) DEFAULT 0,
    latest_run_time TIMESTAMP(6) WITH TIME ZONE,
    latest_status VARCHAR2(20),
    pass_rate_percent NUMBER(5,2),
    has_passed_week NUMBER(1) DEFAULT 0,
    test_run_status VARCHAR2(20),
    last_updated TIMESTAMP(6) WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (test_class_id, tag, week_start_date)
) TABLESPACE QAPORTAL_DATA;

-- Indexes for performance
CREATE INDEX idx_summary_week ON TEST_CLASS_SUMMARY(week_start_date);
CREATE INDEX idx_summary_class ON TEST_CLASS_SUMMARY(test_class_id);
CREATE INDEX idx_summary_status ON TEST_CLASS_SUMMARY(test_run_status);
CREATE INDEX idx_summary_updated ON TEST_CLASS_SUMMARY(last_updated);


-- Populate summary for the last 52 weeks (1 year of history)
INSERT INTO TEST_CLASS_SUMMARY (
    test_class_id,
    test_class_name,
    tag,
    week_start_date,
    week_end_date,
    passed_count,
    total_count,
    latest_run_time,
    latest_status,
    pass_rate_percent,
    has_passed_week,
    test_run_status
)
WITH weekly_ranges AS (
    -- Generate all weeks for the last year
    SELECT 
        TRUNC(SYSDATE, 'IW') - (LEVEL - 1) * 7 AS week_start,
        TRUNC(SYSDATE, 'IW') - (LEVEL - 1) * 7 + 6 + (23/24) + (59/1440) + (59/86400) AS week_end
    FROM DUAL
    CONNECT BY LEVEL <= 52
),
test_runs_by_week AS (
    SELECT
        tc.ID AS test_class_id,
        tc.NAME AS test_class_name,
        NVL(tt.TAG, 'No Tag') AS tag,
        wr.week_start,
        wr.week_end,
        tr.STATUS,
        tr.START_TIME,
        ROW_NUMBER() OVER (
            PARTITION BY tc.ID, NVL(tt.TAG, 'No Tag'), wr.week_start
            ORDER BY tr.START_TIME DESC
        ) AS rn_latest
    FROM weekly_ranges wr
    CROSS JOIN APPLICATION a
    CROSS JOIN TEST_CLASS tc
    CROSS JOIN TEST t
    LEFT JOIN TEST_RUN tr ON tr.TEST_ID = t.ID 
        AND tr.START_TIME BETWEEN wr.week_start AND wr.week_end
    LEFT JOIN TEST_LAUNCH tl ON tr.TEST_LAUNCH_ID = tl.ID
    LEFT JOIN TEST_TAG tt ON tt.TEST_ID = t.ID AND tt.TEST_LAUNCH_ID = tl.ID
    WHERE tc.APP_ID = a.ID
      AND t.TEST_CLASS_ID = tc.ID
      AND (tl.REGRESSION = 1 OR tl.REGRESSION IS NULL)
),
aggregated AS (
    SELECT
        test_class_id,
        test_class_name,
        tag,
        week_start,
        week_end,
        COUNT(CASE WHEN STATUS = 'PASSED' THEN 1 END) AS passed_count,
        COUNT(STATUS) AS total_count,
        MAX(CASE WHEN rn_latest = 1 THEN START_TIME END) AS latest_run_time,
        MAX(CASE WHEN rn_latest = 1 THEN STATUS END) AS latest_status
    FROM test_runs_by_week
    WHERE START_TIME IS NOT NULL  -- Only include weeks with actual runs
    GROUP BY test_class_id, test_class_name, tag, week_start, week_end
)
SELECT
    test_class_id,
    test_class_name,
    tag,
    week_start AS week_start_date,
    week_end AS week_end_date,
    passed_count,
    total_count,
    latest_run_time,
    latest_status,
    CASE 
        WHEN total_count > 0 
        THEN ROUND((passed_count * 100.0) / total_count, 2)
        ELSE 0 
    END AS pass_rate_percent,
    CASE WHEN passed_count > 0 THEN 1 ELSE 0 END AS has_passed_week,
    'Active' AS test_run_status  -- All historical records are active for their week
FROM aggregated
WHERE total_count > 0;  -- Only insert weeks with actual test runs

COMMIT;

```
