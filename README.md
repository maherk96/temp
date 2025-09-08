```sql
DECLARE
    v_test_launch_count NUMBER := 0;
    v_test_step_run_count NUMBER := 0;
    v_test_run_count NUMBER := 0;
    v_test_param_count NUMBER := 0;
    v_test_step_data_count NUMBER := 0;
    v_test_tag_count NUMBER := 0;
    v_performance_metrics_count NUMBER := 0;
    v_exception_count NUMBER := 0;
    v_fix_count NUMBER := 0;
    v_log_count NUMBER := 0;
    
BEGIN
    -- Enable output for this session
    DBMS_OUTPUT.ENABLE(1000000);
    
    DBMS_OUTPUT.PUT_LINE('Starting deletion of non-regression test launches and related data...');
    DBMS_OUTPUT.PUT_LINE('Transaction started at: ' || TO_CHAR(SYSDATE, 'YYYY-MM-DD HH24:MI:SS'));
    
    -- First, let's count what we're about to delete
    SELECT COUNT(*) INTO v_test_launch_count 
    FROM QAPORTAL.TEST_LAUNCH 
    WHERE REGRESSION = 0;
    
    DBMS_OUTPUT.PUT_LINE('Found ' || v_test_launch_count || ' non-regression test launches to delete');
    
    IF v_test_launch_count = 0 THEN
        DBMS_OUTPUT.PUT_LINE('No non-regression test launches found. Exiting.');
        RETURN;
    END IF;
    
    -- Step 1: Delete TEST_STEP_RUN records (must be deleted first due to FK to TEST_RUN)
    DELETE FROM QAPORTAL.TEST_STEP_RUN 
    WHERE TEST_RUN_ID IN (
        SELECT tr.ID 
        FROM QAPORTAL.TEST_RUN tr
        JOIN QAPORTAL.TEST_LAUNCH tl ON tr.TEST_LAUNCH_ID = tl.ID
        WHERE tl.REGRESSION = 0
    );
    v_test_step_run_count := SQL%ROWCOUNT;
    DBMS_OUTPUT.PUT_LINE('Deleted ' || v_test_step_run_count || ' TEST_STEP_RUN records');
    
    -- Step 2: Delete TEST_RUN records
    DELETE FROM QAPORTAL.TEST_RUN 
    WHERE TEST_LAUNCH_ID IN (
        SELECT ID FROM QAPORTAL.TEST_LAUNCH WHERE REGRESSION = 0
    );
    v_test_run_count := SQL%ROWCOUNT;
    DBMS_OUTPUT.PUT_LINE('Deleted ' || v_test_run_count || ' TEST_RUN records');
    
    -- Step 3: Delete TEST_PARAM records
    DELETE FROM QAPORTAL.TEST_PARAM 
    WHERE TEST_LAUNCH_ID IN (
        SELECT ID FROM QAPORTAL.TEST_LAUNCH WHERE REGRESSION = 0
    );
    v_test_param_count := SQL%ROWCOUNT;
    DBMS_OUTPUT.PUT_LINE('Deleted ' || v_test_param_count || ' TEST_PARAM records');
    
    -- Step 4: Delete TEST_STEP_DATA records
    DELETE FROM QAPORTAL.TEST_STEP_DATA 
    WHERE TEST_LAUNCH_ID IN (
        SELECT ID FROM QAPORTAL.TEST_LAUNCH WHERE REGRESSION = 0
    );
    v_test_step_data_count := SQL%ROWCOUNT;
    DBMS_OUTPUT.PUT_LINE('Deleted ' || v_test_step_data_count || ' TEST_STEP_DATA records');
    
    -- Step 5: Delete TEST_TAG records
    DELETE FROM QAPORTAL.TEST_TAG 
    WHERE TEST_LAUNCH_ID IN (
        SELECT ID FROM QAPORTAL.TEST_LAUNCH WHERE REGRESSION = 0
    );
    v_test_tag_count := SQL%ROWCOUNT;
    DBMS_OUTPUT.PUT_LINE('Deleted ' || v_test_tag_count || ' TEST_TAG records');
    
    -- Step 6: Delete PERFORMANCE_METRICS records (if they reference test launches)
    DELETE FROM QAPORTAL.PERFORMANCE_METRICS 
    WHERE TEST_LAUNCH_ID IN (
        SELECT LAUNCH_ID FROM QAPORTAL.TEST_LAUNCH WHERE REGRESSION = 0 AND LAUNCH_ID IS NOT NULL
    );
    v_performance_metrics_count := SQL%ROWCOUNT;
    DBMS_OUTPUT.PUT_LINE('Deleted ' || v_performance_metrics_count || ' PERFORMANCE_METRICS records');
    
    -- Step 7: Delete the TEST_LAUNCH records themselves
    DELETE FROM QAPORTAL.TEST_LAUNCH WHERE REGRESSION = 0;
    DBMS_OUTPUT.PUT_LINE('Deleted ' || v_test_launch_count || ' TEST_LAUNCH records');
    
    -- Step 8: Clean up orphaned EXCEPTION records
    DELETE FROM QAPORTAL.EXCEPTION 
    WHERE ID NOT IN (
        SELECT DISTINCT EXCEPTION_ID 
        FROM QAPORTAL.TEST_RUN 
        WHERE EXCEPTION_ID IS NOT NULL
        UNION
        SELECT DISTINCT EXCEPTION_ID 
        FROM QAPORTAL.TEST_STEP_RUN 
        WHERE EXCEPTION_ID IS NOT NULL
    );
    v_exception_count := SQL%ROWCOUNT;
    DBMS_OUTPUT.PUT_LINE('Deleted ' || v_exception_count || ' orphaned EXCEPTION records');
    
    -- Step 9: Clean up orphaned FIX records
    DELETE FROM QAPORTAL.FIX 
    WHERE ID NOT IN (
        SELECT DISTINCT FIX_ID 
        FROM QAPORTAL.TEST_RUN 
        WHERE FIX_ID IS NOT NULL
        UNION
        SELECT DISTINCT FIX_ID 
        FROM QAPORTAL.TEST_STEP_RUN 
        WHERE FIX_ID IS NOT NULL
    );
    v_fix_count := SQL%ROWCOUNT;
    DBMS_OUTPUT.PUT_LINE('Deleted ' || v_fix_count || ' orphaned FIX records');
    
    -- Step 10: Clean up orphaned LOG records
    DELETE FROM QAPORTAL.LOG 
    WHERE ID NOT IN (
        SELECT DISTINCT LOG_ID 
        FROM QAPORTAL.TEST_RUN 
        WHERE LOG_ID IS NOT NULL
        UNION
        SELECT DISTINCT LOG_ID 
        FROM QAPORTAL.TEST_STEP_RUN 
        WHERE LOG_ID IS NOT NULL
    );
    v_log_count := SQL%ROWCOUNT;
    DBMS_OUTPUT.PUT_LINE('Deleted ' || v_log_count || ' orphaned LOG records');
    
    -- Summary
    DBMS_OUTPUT.PUT_LINE('');
    DBMS_OUTPUT.PUT_LINE('=== DELETION SUMMARY ===');
    DBMS_OUTPUT.PUT_LINE('TEST_LAUNCH: ' || v_test_launch_count);
    DBMS_OUTPUT.PUT_LINE('TEST_STEP_RUN: ' || v_test_step_run_count);
    DBMS_OUTPUT.PUT_LINE('TEST_RUN: ' || v_test_run_count);
    DBMS_OUTPUT.PUT_LINE('TEST_PARAM: ' || v_test_param_count);
    DBMS_OUTPUT.PUT_LINE('TEST_STEP_DATA: ' || v_test_step_data_count);
    DBMS_OUTPUT.PUT_LINE('TEST_TAG: ' || v_test_tag_count);
    DBMS_OUTPUT.PUT_LINE('PERFORMANCE_METRICS: ' || v_performance_metrics_count);
    DBMS_OUTPUT.PUT_LINE('Orphaned EXCEPTION: ' || v_exception_count);
    DBMS_OUTPUT.PUT_LINE('Orphaned FIX: ' || v_fix_count);
    DBMS_OUTPUT.PUT_LINE('Orphaned LOG: ' || v_log_count);
    DBMS_OUTPUT.PUT_LINE('');
    DBMS_OUTPUT.PUT_LINE('Deletion completed successfully at: ' || TO_CHAR(SYSDATE, 'YYYY-MM-DD HH24:MI:SS'));
    
    -- Commit the transaction
    COMMIT;
    DBMS_OUTPUT.PUT_LINE('Transaction committed.');
    
EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Error occurred: ' || SQLERRM);
        DBMS_OUTPUT.PUT_LINE('Rolling back transaction...');
        ROLLBACK;
        RAISE;
END;
/



```
