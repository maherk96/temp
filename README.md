```java

WITH input_params AS (
    SELECT
        'TradingSystem' AS app_name,
        TO_TIMESTAMP('2025-10-20 00:00:00', 'YYYY-MM-DD HH24:MI:SS') AS start_date,
        TO_TIMESTAMP('2025-10-27 23:59:59', 'YYYY-MM-DD HH24:MI:SS') AS end_date
    FROM DUAL
),
base AS (
    SELECT
        a.NAME AS app_name,
        tc.ID AS test_class_id,
        t.ID AS test_id,
        t.METHOD_NAME,
        t.DISPLAY_NAME,
        tr.STATUS,
        tr.START_TIME,
        tl.GIT_BRANCH,
        tl.REGRESSION,
        tt.TAG
    FROM APPLICATION a
    JOIN TEST_CLASS tc ON tc.APP_ID = a.ID
    JOIN TEST t ON t.TEST_CLASS_ID = tc.ID
    LEFT JOIN TEST_RUN tr ON tr.TEST_ID = t.ID
    LEFT JOIN TEST_LAUNCH tl ON tr.TEST_LAUNCH_ID = tl.ID
    LEFT JOIN TEST_TAG tt ON tt.TEST_ID = t.ID AND tt.TEST_LAUNCH_ID = tl.ID
    WHERE a.NAME = (SELECT app_name FROM input_params)
      AND tl.REGRESSION = 1
),
-- All runs within the last 4 weeks (3 prior + current week)
recent_runs AS (
    SELECT *
    FROM base
    WHERE tr.START_TIME BETWEEN (SELECT start_date FROM input_params) - INTERVAL '21' DAY
                             AND (SELECT end_date FROM input_params)
)
SELECT
    r.app_name,
    r.test_id,
    r.display_name,
    r.method_name,
    NVL(r.tag, 'No Tag') AS tag,
    -- All runs within the selected week
    LISTAGG(
        TO_CHAR(r.start_time, 'YYYY-MM-DD HH24:MI') || ' → ' || r.status,
        '; '
    ) WITHIN GROUP (ORDER BY r.start_time) AS runs_this_week,
    -- Most recent run date and status overall (last 4 weeks)
    MAX(r.start_time) AS last_run_time,
    MAX(
        CASE WHEN r.start_time = (
            SELECT MAX(r2.start_time)
            FROM recent_runs r2
            WHERE r2.test_id = r.test_id
              AND NVL(r2.tag, 'No Tag') = NVL(r.tag, 'No Tag')
        )
        THEN r.status END
    ) AS last_run_status,
    -- Pass flag if any run passed in current period
    MAX(
        CASE WHEN r.status = 'PASSED'
             AND r.start_time BETWEEN (SELECT start_date FROM input_params)
                                   AND (SELECT end_date FROM input_params)
             THEN 1 ELSE 0 END
    ) AS has_passed_this_week,
    -- Stale logic
    CASE
        WHEN MAX(CASE WHEN r.start_time BETWEEN (SELECT start_date FROM input_params)
                                           AND (SELECT end_date FROM input_params)
                      THEN 1 END) = 1
        THEN 'Active'
        WHEN MAX(CASE WHEN r.start_time BETWEEN (SELECT start_date FROM input_params) - INTERVAL '21' DAY
                                           AND (SELECT start_date FROM input_params)
                      THEN 1 END) = 1
        THEN 'Stale'
        ELSE 'Never Run'
    END AS test_run_status
FROM recent_runs r
GROUP BY r.app_name, r.test_id, r.display_name, r.method_name, r.tag
ORDER BY r.display_name, r.tag;

```
