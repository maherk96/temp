```sql

WITH input_params AS (
    SELECT
        'Fusion Algo' AS app_name,
        TRUNC(TO_DATE('2025-10-21', 'YYYY-MM-DD'), 'IW') AS current_week_start,
        TRUNC(TO_DATE('2025-10-21', 'YYYY-MM-DD'), 'IW') - 7 AS prior_week_start
    FROM DUAL
)
SELECT
    s.test_class_id,
    s.test_class_name,
    s.tag,
    s.latest_run_time AS last_run_time,
    s.latest_status AS last_run_status,
    s.passed_count AS passed_this_week,
    s.total_count AS total_runs_this_week,
    s.pass_rate_percent,
    s.has_passed_week AS has_passed_this_week,
    CASE 
        WHEN s.week_start_date = (SELECT current_week_start FROM input_params) 
        THEN 'Active'
        WHEN EXISTS (
            SELECT 1 FROM TEST_CLASS_SUMMARY s2
            WHERE s2.test_class_id = s.test_class_id
              AND s2.tag = s.tag
              AND s2.week_start_date BETWEEN 
                  (SELECT current_week_start FROM input_params) - 21
                  AND (SELECT prior_week_start FROM input_params)
        )
        THEN 'Stale'
        ELSE 'Never Run'
    END AS test_run_status
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
ORDER BY s.test_class_name, s.tag, s.week_start_date DESC;

```
