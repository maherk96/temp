```sql

WITH input_params AS (
    SELECT
        'Fusion Algo' AS app_name,
        TO_TIMESTAMP('2025-10-21 00:00:00', 'YYYY-MM-DD HH24:MI:SS') AS start_date,
        TO_TIMESTAMP('2025-10-27 23:59:59', 'YYYY-MM-DD HH24:MI:SS') AS end_date
    FROM DUAL
),
base_filtered AS (
    SELECT
        a.NAME AS app_name,
        tc.ID AS test_class_id,
        tc.NAME AS test_class_name,
        tr.STATUS,
        tr.START_TIME,
        tl.REGRESSION,
        NVL(tt.TAG, 'No Tag') AS tag,
        ROW_NUMBER() OVER (
            PARTITION BY tc.ID, NVL(tt.TAG, 'No Tag')
            ORDER BY tr.START_TIME DESC
        ) AS rn_latest_run
    FROM APPLICATION a
    JOIN TEST_CLASS tc ON tc.APP_ID = a.ID
    JOIN TEST t ON t.TEST_CLASS_ID = tc.ID
    LEFT JOIN TEST_RUN tr ON tr.TEST_ID = t.ID
    LEFT JOIN TEST_LAUNCH tl ON tr.TEST_LAUNCH_ID = tl.ID AND tl.REGRESSION = 1  -- ✅ Filter in JOIN
    LEFT JOIN TEST_TAG tt ON tt.TEST_ID = t.ID AND tt.TEST_LAUNCH_ID = tl.ID
    WHERE a.NAME = (SELECT app_name FROM input_params)
      AND (tr.START_TIME BETWEEN 
              (SELECT start_date FROM input_params) - INTERVAL '21' DAY 
              AND (SELECT end_date FROM input_params)
           OR tr.START_TIME IS NULL)  -- Include never-run tests
),
latest_runs_status AS (
    SELECT
        test_class_id,
        tag,
        STATUS AS latest_status,
        START_TIME AS latest_start_time
    FROM base_filtered
    WHERE rn_latest_run = 1
)
SELECT
    bf.app_name,
    bf.test_class_id,
    bf.test_class_name,
    bf.tag,
    lrs.latest_start_time AS last_run_time,
    lrs.latest_status AS last_run_status,
    
    -- Count of passed tests in current week
    COUNT(CASE 
        WHEN bf.status = 'PASSED' 
         AND bf.start_time BETWEEN (SELECT start_date FROM input_params) 
                               AND (SELECT end_date FROM input_params)
        THEN 1 
    END) AS passed_this_week,
    
    -- Total test runs in current week
    COUNT(CASE 
        WHEN bf.start_time BETWEEN (SELECT start_date FROM input_params) 
                               AND (SELECT end_date FROM input_params)
        THEN 1 
    END) AS total_runs_this_week,
    
    -- Pass rate percentage
    CASE 
        WHEN COUNT(CASE 
                WHEN bf.start_time BETWEEN (SELECT start_date FROM input_params) 
                                       AND (SELECT end_date FROM input_params)
                THEN 1 
             END) > 0
        THEN ROUND(
            COUNT(CASE 
                WHEN bf.status = 'PASSED' 
                 AND bf.start_time BETWEEN (SELECT start_date FROM input_params) 
                                       AND (SELECT end_date FROM input_params)
                THEN 1 
            END) * 100.0 / 
            COUNT(CASE 
                WHEN bf.start_time BETWEEN (SELECT start_date FROM input_params) 
                                       AND (SELECT end_date FROM input_params)
                THEN 1 
            END), 
            2
        )
        ELSE 0
    END AS pass_rate_percent,
    
    -- Has at least one pass this week flag
    MAX(
        CASE WHEN bf.status = 'PASSED'
             AND bf.start_time BETWEEN (SELECT start_date FROM input_params) 
                                   AND (SELECT end_date FROM input_params)
        THEN 1 ELSE 0 END
    ) AS has_passed_this_week,
    
    -- Activity status
    CASE
        WHEN MAX(CASE 
                WHEN bf.start_time BETWEEN (SELECT start_date FROM input_params) 
                                       AND (SELECT end_date FROM input_params) 
                THEN 1 
             END) = 1 
        THEN 'Active'
        WHEN MAX(CASE 
                WHEN bf.start_time BETWEEN (SELECT start_date FROM input_params) - INTERVAL '21' DAY 
                                       AND (SELECT start_date FROM input_params) 
                THEN 1 
             END) = 1 
        THEN 'Stale'
        ELSE 'Never Run'
    END AS test_run_status
    
FROM base_filtered bf
LEFT JOIN latest_runs_status lrs
    ON bf.test_class_id = lrs.test_class_id
   AND bf.tag = lrs.tag
GROUP BY 
    bf.app_name, 
    bf.test_class_id, 
    bf.test_class_name, 
    bf.tag, 
    lrs.latest_start_time, 
    lrs.latest_status
ORDER BY bf.test_class_name, bf.tag;

```
