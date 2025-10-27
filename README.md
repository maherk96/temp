```sql
WITH input_params AS (
    SELECT
        'TestApp' AS app_name,
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
        NVL(tt.TAG, 'No Tag') AS tag,
        ROW_NUMBER() OVER (
            PARTITION BY tc.ID, NVL(tt.TAG, 'No Tag')
            ORDER BY tr.START_TIME DESC
        ) AS rn_latest_run
    FROM APPLICATION a
    JOIN TEST_CLASS tc ON tc.APP_ID = a.ID
    JOIN TEST t ON t.TEST_CLASS_ID = tc.ID
    JOIN TEST_RUN tr ON tr.TEST_ID = t.ID
    JOIN TEST_LAUNCH tl ON tr.TEST_LAUNCH_ID = tl.ID
    LEFT JOIN TEST_TAG tt ON tt.TEST_ID = t.ID AND tt.TEST_LAUNCH_ID = tl.ID
    WHERE a.NAME = (SELECT app_name FROM input_params)
      AND tl.REGRESSION = 1
      AND tr.START_TIME BETWEEN 
              (SELECT start_date FROM input_params) - INTERVAL '21' DAY 
              AND (SELECT end_date FROM input_params)
),
latest_runs_status AS (
    SELECT
        test_class_id,
        tag,
        STATUS AS latest_status,
        START_TIME AS latest_start_time
    FROM base_filtered
    WHERE rn_latest_run = 1
),
current_week_stats AS (
    -- Stats for tests that ran THIS week
    SELECT
        bf.test_class_id,
        bf.test_class_name,
        bf.tag,
        COUNT(CASE WHEN bf.status = 'PASSED' THEN 1 END) AS passed_this_week,
        COUNT(*) AS total_runs_this_week,
        1 AS ran_this_week
    FROM base_filtered bf
    CROSS JOIN input_params p
    WHERE bf.start_time BETWEEN p.start_date AND p.end_date
    GROUP BY bf.test_class_id, bf.test_class_name, bf.tag
),
stale_stats AS (
    -- For stale tests, get their stats from when they LAST ran
    SELECT
        bf.test_class_id,
        bf.test_class_name,
        bf.tag,
        COUNT(CASE WHEN bf.status = 'PASSED' THEN 1 END) AS passed_last_run,
        COUNT(*) AS total_runs_last_run
    FROM base_filtered bf
    WHERE bf.rn_latest_run <= (
        -- Get all runs from the same launch as the latest run
        SELECT COUNT(*)
        FROM base_filtered bf2
        WHERE bf2.test_class_id = bf.test_class_id
          AND bf2.tag = bf.tag
          AND DATE_TRUNC('day', bf2.start_time) = DATE_TRUNC('day', (
              SELECT MAX(bf3.start_time)
              FROM base_filtered bf3
              WHERE bf3.test_class_id = bf.test_class_id
                AND bf3.tag = bf.tag
          ))
    )
    GROUP BY bf.test_class_id, bf.test_class_name, bf.tag
)
SELECT
    (SELECT app_name FROM input_params) AS app_name,
    COALESCE(cws.test_class_id, ss.test_class_id) AS test_class_id,
    COALESCE(cws.test_class_name, ss.test_class_name) AS test_class_name,
    COALESCE(cws.tag, ss.tag) AS tag,
    lrs.latest_start_time AS last_run_time,
    lrs.latest_status AS last_run_status,
    
    -- Use current week stats if available, otherwise use stale stats
    COALESCE(cws.passed_this_week, ss.passed_last_run, 0) AS passed_this_week,
    COALESCE(cws.total_runs_this_week, ss.total_runs_last_run, 0) AS total_runs_this_week,
    
    -- Pass rate percentage
    CASE 
        WHEN COALESCE(cws.total_runs_this_week, ss.total_runs_last_run, 0) > 0 
        THEN ROUND(
            COALESCE(cws.passed_this_week, ss.passed_last_run, 0) * 100.0 / 
            COALESCE(cws.total_runs_this_week, ss.total_runs_last_run, 0), 
            2
        )
        ELSE 0 
    END AS pass_rate_percent,
    
    -- Has at least one pass
    CASE WHEN COALESCE(cws.passed_this_week, ss.passed_last_run, 0) > 0 THEN 1 ELSE 0 END AS has_passed_this_week,
    
    -- Activity status
    CASE
        WHEN cws.ran_this_week = 1 THEN 'Active'
        ELSE 'Stale'
    END AS test_run_status
    
FROM latest_runs_status lrs
LEFT JOIN current_week_stats cws
    ON lrs.test_class_id = cws.test_class_id
   AND lrs.tag = cws.tag
LEFT JOIN stale_stats ss
    ON lrs.test_class_id = ss.test_class_id
   AND lrs.tag = ss.tag
ORDER BY 
    COALESCE(cws.test_class_name, ss.test_class_name), 
    COALESCE(cws.tag, ss.tag);


```
