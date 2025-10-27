```java

WITH input_params AS (
    SELECT
        'TradingSystem' AS app_name,
        TO_TIMESTAMP('2025-10-20 00:00:00', 'YYYY-MM-DD HH24:MI:SS') AS start_date,
        TO_TIMESTAMP('2025-10-27 23:59:59', 'YYYY-MM-DD HH24:MI:SS') AS end_date
    FROM DUAL
)
SELECT
    a.NAME AS app_name,
    t.ID AS test_id,
    t.METHOD_NAME,
    t.DISPLAY_NAME,
    NVL(tt.TAG, 'No Tag') AS tag,
    tr.STATUS,
    tr.START_TIME,
    tl.REGRESSION
FROM APPLICATION a
JOIN TEST_CLASS tc ON tc.APP_ID = a.ID
JOIN TEST t ON t.TEST_CLASS_ID = tc.ID
JOIN TEST_RUN tr ON tr.TEST_ID = t.ID
JOIN TEST_LAUNCH tl ON tr.TEST_LAUNCH_ID = tl.ID
LEFT JOIN TEST_TAG tt ON tt.TEST_ID = t.ID AND tt.TEST_LAUNCH_ID = tl.ID
WHERE a.NAME = (SELECT app_name FROM input_params)
  AND tl.REGRESSION = 1
  AND tr.START_TIME BETWEEN (SELECT start_date FROM input_params) - INTERVAL '21' DAY
                        AND (SELECT end_date FROM input_params)
ORDER BY t.DISPLAY_NAME, tr.START_TIME;

```
