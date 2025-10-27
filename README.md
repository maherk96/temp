```SQL
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
        tc.NAME AS test_class_name,
        t.ID AS test_id,
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
recent_runs AS (
    SELECT *
    FROM base
    WHERE START_TIME BETWEEN (SELECT start_date FROM input_params) - INTERVAL '21' DAY
                         AND (SELECT end_date FROM input_params)
)
SELECT
    r.app_name,
    r.test_class_id,
    r.test_class_name,
    NVL(r.tag, 'No Tag') AS tag,

    -- All runs (for any test in the class) within the selected week
    LISTAGG(
        TO_CHAR(r.start_time, 'YYYY-MM-DD HH24:MI') || ' → ' || r.status,
        '; '
    ) WITHIN GROUP (ORDER BY r.start_time) AS runs_this_week,

    -- Most recent run timestamp across the class/tag combo
    MAX(r.start_time) AS last_run_time,

    -- Last run status (for that class/tag combo)
    MAX(
        CASE WHEN r.start_time = (
            SELECT MAX(r2.start_time)
            FROM recent_runs r2
            WHERE r2.test_class_id = r.test_class_id
              AND NVL(r2.tag, 'No Tag') = NVL(r.tag, 'No Tag')
        )
        THEN r.status END
    ) AS last_run_status,

    -- Flag: did this tag (within class) pass at least once this week?
    MAX(
        CASE WHEN r.status = 'PASSED'
             AND r.start_time BETWEEN (SELECT start_date FROM input_params)
                                   AND (SELECT end_date FROM input_params)
             THEN 1 ELSE 0 END
    ) AS has_passed_this_week,

    -- Determine if tag was run, stale, or never run (class-level)
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
    END AS tag_run_status
FROM recent_runs r
GROUP BY r.app_name, r.test_class_id, r.test_class_name, r.tag
ORDER BY r.test_class_name, r.tag;
```
