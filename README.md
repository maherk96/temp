```sql
WITH input_params AS (
    SELECT
        'Fusion Algo' AS app_name,
        TRUNC(TO_DATE('2025-10-21', 'YYYY-MM-DD'), 'IW') AS current_week_start,
        TRUNC(TO_DATE('2025-10-27', 'YYYY-MM-DD'), 'IW') AS current_week_end
    FROM DUAL
),
aggregated_data AS (
    -- Aggregate across all 4 weeks to get one row per test_class + tag
    SELECT
        s.test_class_id,
        s.test_class_name,
        s.tag,
        -- Latest run across all 4 weeks
        MAX(s.latest_run_time) AS last_run_time,
        MAX(s.latest_status) KEEP (DENSE_RANK LAST ORDER BY s.latest_run_time) AS last_run_status,
        -- Sum passed/total from CURRENT week only
        SUM(CASE WHEN s.week_start_date = (SELECT current_week_start FROM input_params) 
                 THEN s.passed_count ELSE 0 END) AS passed_this_week,
        SUM(CASE WHEN s.week_start_date = (SELECT current_week_start FROM input_params) 
                 THEN s.total_count ELSE 0 END) AS total_runs_this_week,
        -- Has passed flag for current week
        MAX(CASE WHEN s.week_start_date = (SELECT current_week_start FROM input_params) 
                 THEN s.has_passed_week ELSE 0 END) AS has_passed_this_week,
        -- Check if ran in current week
        MAX(CASE WHEN s.week_start_date = (SELECT current_week_start FROM input_params) 
                 THEN 1 ELSE 0 END) AS ran_current_week
    FROM TEST_CLASS_SUMMARY s
    CROSS JOIN input_params p
    WHERE s.week_start_date >= p.current_week_start - 21  -- Last 4 weeks
      AND s.week_start_date <= p.current_week_start
      AND EXISTS (
          SELECT 1 
          FROM TEST_CLASS tc 
          JOIN APPLICATION a ON tc.APP_ID = a.ID
          WHERE tc.ID = s.test_class_id 
            AND a.NAME = p.app_name
      )
    GROUP BY s.test_class_id, s.test_class_name, s.tag
)
SELECT
    (SELECT app_name FROM input_params) AS app_name,
    test_class_id,
    test_class_name,
    tag,
    last_run_time,
    last_run_status,
    passed_this_week,
    total_runs_this_week,
    CASE 
        WHEN total_runs_this_week > 0 
        THEN ROUND((passed_this_week * 100.0) / total_runs_this_week, 2)
        ELSE 0 
    END AS pass_rate_percent,
    has_passed_this_week,
    CASE
        WHEN ran_current_week = 1 THEN 'Active'
        ELSE 'Stale'
    END AS test_run_status
FROM aggregated_data
ORDER BY test_class_name, tag;


```
