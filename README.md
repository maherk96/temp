```sql

  WITH LatestTestRunInLast4Weeks AS (
    SELECT 
        MAX(tr.ID) AS latest_run_id,
        tr.TEST_ID
    FROM QAPORTAL.TEST_LAUNCH tl
    INNER JOIN QAPORTAL.TEST_RUN tr ON tl.ID = tr.TEST_LAUNCH_ID
    INNER JOIN QAPORTAL.APPLICATION a ON tl.APP_ID = a.ID
    WHERE a.NAME = ?
      AND a.REGRESSION = 1
      AND tl.CREATED BETWEEN TO_DATE(?, 'YYYY-MM-DD HH24:MI:SS') 
                         AND TO_DATE(?, 'YYYY-MM-DD HH24:MI:SS')
      AND tr.STATUS IN ('PASSED', 'FAILED')
    GROUP BY tr.TEST_ID
  )
  SELECT
      tc.NAME AS testClassName,
      tt.TAG AS tag,
      SUM(CASE WHEN tr.STATUS != 'FAILED' THEN 1 ELSE 0 END) AS passed,
      COUNT(tr.STATUS) AS total,
      MAX(tr.START_TIME) AS lastRun
  FROM LatestTestRunInLast4Weeks ltr
  INNER JOIN QAPORTAL.TEST_RUN tr ON ltr.latest_run_id = tr.ID
  INNER JOIN QAPORTAL.TEST t ON tr.TEST_ID = t.ID
  INNER JOIN QAPORTAL.TEST_CLASS tc ON t.TEST_CLASS_ID = tc.ID
  INNER JOIN QAPORTAL.TEST_TAG tt ON t.ID = tt.TEST_ID 
                                  AND tr.TEST_LAUNCH_ID = tt.TEST_LAUNCH_ID
  GROUP BY tc.NAME, tt.TAG
  ORDER BY testClassName, tag
```
