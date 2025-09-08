```sql
DECLARE
    v_batch_size NUMBER := 1000;  -- Process in smaller batches
    v_total_deleted NUMBER := 0;
    v_batch_deleted NUMBER := 1;
    
BEGIN
    -- Enable output for this session
    DBMS_OUTPUT.ENABLE(1000000);
    
    DBMS_OUTPUT.PUT_LINE('Starting optimized deletion in batches of ' || v_batch_size);
    DBMS_OUTPUT.PUT_LINE('Started at: ' || TO_CHAR(SYSDATE, 'YYYY-MM-DD HH24:MI:SS'));
    
    -- Step 1: Delete TEST_STEP_RUN in batches
    DBMS_OUTPUT.PUT_LINE('Deleting TEST_STEP_RUN records...');
    WHILE v_batch_deleted > 0 LOOP
        DELETE FROM QAPORTAL.TEST_STEP_RUN 
        WHERE TEST_RUN_ID IN (
            SELECT tr.ID 
            FROM QAPORTAL.TEST_RUN tr
            JOIN QAPORTAL.TEST_LAUNCH tl ON tr.TEST_LAUNCH_ID = tl.ID
            WHERE tl.REGRESSION = 0
            AND ROWNUM <= v_batch_size
        );
        v_batch_deleted := SQL%ROWCOUNT;
        v_total_deleted := v_total_deleted + v_batch_deleted;
        
        IF MOD(v_total_deleted, 5000) = 0 THEN
            DBMS_OUTPUT.PUT_LINE('Deleted ' || v_total_deleted || ' TEST_STEP_RUN records so far...');
            COMMIT; -- Commit periodically to avoid long transactions
        END IF;
    END LOOP;
    DBMS_OUTPUT.PUT_LINE('Completed TEST_STEP_RUN: ' || v_total_deleted || ' records deleted');
    
    -- Step 2: Delete TEST_RUN in batches
    DBMS_OUTPUT.PUT_LINE('Deleting TEST_RUN records...');
    v_total_deleted := 0;
    v_batch_deleted := 1;
    WHILE v_batch_deleted > 0 LOOP
        DELETE FROM QAPORTAL.TEST_RUN 
        WHERE TEST_LAUNCH_ID IN (
            SELECT ID FROM QAPORTAL.TEST_LAUNCH 
            WHERE REGRESSION = 0
            AND ROWNUM <= v_batch_size
        );
        v_batch_deleted := SQL%ROWCOUNT;
        v_total_deleted := v_total_deleted + v_batch_deleted;
        
        IF MOD(v_total_deleted, 5000) = 0 THEN
            DBMS_OUTPUT.PUT_LINE('Deleted ' || v_total_deleted || ' TEST_RUN records so far...');
            COMMIT;
        END IF;
    END LOOP;
    DBMS_OUTPUT.PUT_LINE('Completed TEST_RUN: ' || v_total_deleted || ' records deleted');
    
    -- Step 3: Delete remaining tables in batches (similar pattern)
    DBMS_OUTPUT.PUT_LINE('Deleting TEST_PARAM records...');
    DELETE FROM QAPORTAL.TEST_PARAM 
    WHERE TEST_LAUNCH_ID IN (
        SELECT ID FROM QAPORTAL.TEST_LAUNCH WHERE REGRESSION = 0
    );
    DBMS_OUTPUT.PUT_LINE('Deleted ' || SQL%ROWCOUNT || ' TEST_PARAM records');
    
    DELETE FROM QAPORTAL.TEST_STEP_DATA 
    WHERE TEST_LAUNCH_ID IN (
        SELECT ID FROM QAPORTAL.TEST_LAUNCH WHERE REGRESSION = 0
    );
    DBMS_OUTPUT.PUT_LINE('Deleted ' || SQL%ROWCOUNT || ' TEST_STEP_DATA records');
    
    DELETE FROM QAPORTAL.TEST_TAG 
    WHERE TEST_LAUNCH_ID IN (
        SELECT ID FROM QAPORTAL.TEST_LAUNCH WHERE REGRESSION = 0
    );
    DBMS_OUTPUT.PUT_LINE('Deleted ' || SQL%ROWCOUNT || ' TEST_TAG records');
    
    DELETE FROM QAPORTAL.PERFORMANCE_METRICS 
    WHERE TEST_LAUNCH_ID IN (
        SELECT LAUNCH_ID FROM QAPORTAL.TEST_LAUNCH 
        WHERE REGRESSION = 0 AND LAUNCH_ID IS NOT NULL
    );
    DBMS_OUTPUT.PUT_LINE('Deleted ' || SQL%ROWCOUNT || ' PERFORMANCE_METRICS records');
    
    -- Step 4: Delete TEST_LAUNCH records
    DELETE FROM QAPORTAL.TEST_LAUNCH WHERE REGRESSION = 0;
    DBMS_OUTPUT.PUT_LINE('Deleted ' || SQL%ROWCOUNT || ' TEST_LAUNCH records');
    
    -- Final cleanup of orphaned records
    DELETE FROM QAPORTAL.EXCEPTION 
    WHERE ID NOT IN (
        SELECT DISTINCT EXCEPTION_ID FROM QAPORTAL.TEST_RUN WHERE EXCEPTION_ID IS NOT NULL
        UNION
        SELECT DISTINCT EXCEPTION_ID FROM QAPORTAL.TEST_STEP_RUN WHERE EXCEPTION_ID IS NOT NULL
    );
    DBMS_OUTPUT.PUT_LINE('Deleted ' || SQL%ROWCOUNT || ' orphaned EXCEPTION records');
    
    DELETE FROM QAPORTAL.FIX 
    WHERE ID NOT IN (
        SELECT DISTINCT FIX_ID FROM QAPORTAL.TEST_RUN WHERE FIX_ID IS NOT NULL
        UNION
        SELECT DISTINCT FIX_ID FROM QAPORTAL.TEST_STEP_RUN WHERE FIX_ID IS NOT NULL
    );
    DBMS_OUTPUT.PUT_LINE('Deleted ' || SQL%ROWCOUNT || ' orphaned FIX records');
    
    DELETE FROM QAPORTAL.LOG 
    WHERE ID NOT IN (
        SELECT DISTINCT LOG_ID FROM QAPORTAL.TEST_RUN WHERE LOG_ID IS NOT NULL
        UNION
        SELECT DISTINCT LOG_ID FROM QAPORTAL.TEST_STEP_RUN WHERE LOG_ID IS NOT NULL
    );
    DBMS_OUTPUT.PUT_LINE('Deleted ' || SQL%ROWCOUNT || ' orphaned LOG records');
    
    COMMIT;
    DBMS_OUTPUT.PUT_LINE('All deletions completed successfully at: ' || TO_CHAR(SYSDATE, 'YYYY-MM-DD HH24:MI:SS'));
    
EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
        ROLLBACK;
        RAISE;
END;

```
