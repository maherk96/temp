```sql
WITH input_params AS (
    SELECT
        'TestApp' AS app_name,
        TO_TIMESTAMP('2025-10-21 00:00:00', 'YYYY-MM-DD HH24:MI:SS') AS start_date,
        TO_TIMESTAMP('2025-10-27 23:59:59', 'YYYY-MM-DD HH24:MI:SS') AS end_date
    FROM DUAL
),
-- Single scan of all relevant test runs in the 4-week window
all_runs AS (
    SELECT
        tc.ID AS test_class_id,
        tc.NAME AS test_class_name,
        NVL(tt.TAG, 'No Tag') AS tag,
        tr.STATUS,
        tr.START_TIME,
        tr.TEST_LAUNCH_ID,
        -- Flag if run is in current week (avoids repeated date comparisons)
        CASE 
            WHEN tr.START_TIME >= (SELECT start_date FROM input_params)
             AND tr.START_TIME <= (SELECT end_date FROM input_params)
            THEN 1 
            ELSE 0 
        END AS is_current_week,
        -- Single ROW_NUMBER for latest run per class+tag
        ROW_NUMBER() OVER (
            PARTITION BY tc.ID, NVL(tt.TAG, 'No Tag')
            ORDER BY tr.START_TIME DESC
        ) AS rn_latest_overall
    FROM APPLICATION a
    JOIN TEST_CLASS tc ON tc.APP_ID = a.ID
    JOIN TEST t ON t.TEST_CLASS_ID = tc.ID
    JOIN TEST_RUN tr ON tr.TEST_ID = t.ID
    JOIN TEST_LAUNCH tl ON tr.TEST_LAUNCH_ID = tl.ID
    LEFT JOIN TEST_TAG tt ON tt.TEST_ID = t.ID AND tt.TEST_LAUNCH_ID = tl.ID
    WHERE a.NAME = (SELECT app_name FROM input_params)
      AND tl.REGRESSION = 1
      AND tr.START_TIME >= (SELECT start_date FROM input_params) - INTERVAL '21' DAY
      AND tr.START_TIME <= (SELECT end_date FROM input_params)
),
-- Aggregate current week stats (single group by)
current_week_agg AS (
    SELECT
        test_class_id,
        test_class_name,
        tag,
        COUNT(CASE WHEN STATUS = 'PASSED' THEN 1 END) AS passed_count,
        COUNT(*) AS total_count
    FROM all_runs
    WHERE is_current_week = 1
    GROUP BY test_class_id, test_class_name, tag
),
-- Get the latest run info (single filter on rn = 1)
latest_run_info AS (
    SELECT
        test_class_id,
        test_class_name,
        tag,
        START_TIME AS last_run_time,
        STATUS AS last_run_status,
        TEST_LAUNCH_ID AS last_launch_id,
        is_current_week AS last_run_in_current_week
    FROM all_runs
    WHERE rn_latest_overall = 1
),
-- For stale tests only: get stats from their last launch
stale_last_launch_agg AS (
    SELECT
        ar.test_class_id,
        ar.tag,
        COUNT(CASE WHEN ar.STATUS = 'PASSED' THEN 1 END) AS passed_count,
        COUNT(*) AS total_count
    FROM all_runs ar
    INNER JOIN latest_run_info lri
        ON ar.test_class_id = lri.test_class_id
       AND ar.tag = lri.tag
       AND ar.TEST_LAUNCH_ID = lri.last_launch_id
    WHERE lri.last_run_in_current_week = 0  -- Only for stale tests
    GROUP BY ar.test_class_id, ar.tag
)
SELECT
    (SELECT app_name FROM input_params) AS app_name,
    lri.test_class_id,
    lri.test_class_name,
    lri.tag,
    lri.last_run_time,
    lri.last_run_status,
    
    -- Use current week stats if available, otherwise stale launch stats
    COALESCE(cwa.passed_count, sla.passed_count, 0) AS passed_this_week,
    COALESCE(cwa.total_count, sla.total_count, 0) AS total_runs_this_week,
    
    -- Pass rate calculation
    CASE 
        WHEN COALESCE(cwa.total_count, sla.total_count, 0) > 0 
        THEN ROUND(
            COALESCE(cwa.passed_count, sla.passed_count, 0) * 100.0 / 
            COALESCE(cwa.total_count, sla.total_count, 0), 
            2
        )
        ELSE 0 
    END AS pass_rate_percent,
    
    -- Has passed flag
    CASE 
        WHEN COALESCE(cwa.passed_count, sla.passed_count, 0) > 0 
        THEN 1 
        ELSE 0 
    END AS has_passed_this_week,
    
    -- Activity status
    CASE
        WHEN lri.last_run_in_current_week = 1 THEN 'Active'
        ELSE 'Stale'
    END AS test_run_status
    
FROM latest_run_info lri
LEFT JOIN current_week_agg cwa
    ON lri.test_class_id = cwa.test_class_id
   AND lri.tag = cwa.tag
LEFT JOIN stale_last_launch_agg sla
    ON lri.test_class_id = sla.test_class_id
   AND lri.tag = sla.tag
ORDER BY lri.test_class_name, lri.tag;


```
