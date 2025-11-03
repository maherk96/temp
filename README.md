```sql
-- TestRunDataQuery (corrected join logic)
SELECT 
    r.CREATED AS created,
    r.START_TIME AS startTime,
    r.STATUS AS status,
    e."EXCEPTION",
    tl.GIT_BRANCH AS gitBranch,
    tl.TEST_RUNNER_VERSION AS testRunnerVersion,
    tl.JDK_VERSION AS jdkVersion,
    tl.OS_VERSION AS osVersion,
    e2.NAME AS envName,
    u.NAME AS userName,
    r.tags
FROM (
    SELECT 
        tr.CREATED,
        tr.TEST_LAUNCH_ID,
        tr.START_TIME,
        tr.STATUS,
        tr.EXCEPTION_ID,
        LISTAGG(DISTINCT tg.NAME, ', ') 
            WITHIN GROUP (ORDER BY tg.NAME) AS tags
    FROM TEST_RUN tr
    INNER JOIN TEST t 
        ON tr.TEST_ID = t.ID
    LEFT JOIN TEST_TAG tt 
        ON (tt.TEST_ID = t.ID OR tt.TEST_LAUNCH_ID = tr.TEST_LAUNCH_ID)
    LEFT JOIN TAG tg 
        ON tg.ID = tt.TAG_ID
    WHERE tr.START_TIME > TRUNC(SYSDATE) - 14
    GROUP BY tr.CREATED, tr.TEST_LAUNCH_ID, tr.START_TIME, tr.STATUS, tr.EXCEPTION_ID
    ORDER BY tr.START_TIME DESC
    FETCH FIRST 5 ROWS ONLY
) r
LEFT JOIN "EXCEPTION" e 
    ON r.EXCEPTION_ID = e.ID
INNER JOIN TEST_LAUNCH tl 
    ON r.TEST_LAUNCH_ID = tl.ID
INNER JOIN ENVIRONMENTS e2 
    ON tl.ENV_ID = e2.ID
INNER JOIN USERS u 
    ON tl.USER_ID = u.ID;
```
