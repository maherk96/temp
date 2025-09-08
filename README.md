```sql

-- 1. Add a new temporary column
ALTER TABLE qaportal.test_feature 
ADD created_temp TIMESTAMP(6) WITH TIME ZONE;

-- 2. Convert and copy data (if it's in a valid timestamp format)
UPDATE qaportal.test_feature 
SET created_temp = TO_TIMESTAMP_TZ(created, 'YYYY-MM-DD HH24:MI:SS.FF TZH:TZM')
WHERE created IS NOT NULL;

-- 3. Drop old column
ALTER TABLE qaportal.test_feature DROP COLUMN created;

-- 4. Rename new column
ALTER TABLE qaportal.test_feature RENAME COLUMN created_temp TO created;

```
