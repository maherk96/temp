```sql

-- Check test links (should have data)
SELECT * FROM coverage_item_test_link WHERE coverage_item_id = YOUR_ITEM_ID;

-- Check test runs (likely empty or outside window)
SELECT tr.*, tl.start_time 
FROM test_run tr
JOIN test_launch tl ON tr.test_launch_id = tl.id
WHERE tr.test_id IN (SELECT test_id FROM coverage_item_test_link WHERE coverage_item_id = YOUR_ITEM_ID)
ORDER BY tl.start_time DESC;

```
