```sql
SELECT t.id,
       t.method_name,
       t.display_name,
       t.test_class_id,
       t.created
FROM test t
WHERE t.method_name = 'spotDuplicateClOrdID'
  AND t.display_name = 'Client order rejected on inception due to supplying duplicate ClOrdID'
ORDER BY t.created DESC;


SELECT method_name,
       display_name,
       test_class_id,
       COUNT(*) AS duplicates
FROM test
WHERE method_name = 'spotDuplicateClOrdID'
  AND display_name = 'Client order rejected on inception due to supplying duplicate ClOrdID'
GROUP BY method_name, display_name, test_class_id
HAVING COUNT(*) > 1;

SELECT method_name,
       display_name,
       test_class_id,
       COUNT(*) AS count
FROM test
GROUP BY method_name, display_name, test_class_id
HAVING COUNT(*) > 1;

SELECT *
FROM (
    SELECT t.*,
           ROW_NUMBER() OVER (
               PARTITION BY method_name, display_name, test_class_id
               ORDER BY created ASC
           ) AS rn
    FROM test t
) x
WHERE x.rn > 1
ORDER BY x.method_name, x.display_name, x.test_class_id, x.created;


DELETE FROM test
WHERE id IN (
    SELECT id FROM (
        SELECT t.id,
               ROW_NUMBER() OVER (
                   PARTITION BY method_name, display_name, test_class_id
                   ORDER BY created ASC
               ) AS rn
        FROM test t
    ) d
    WHERE d.rn > 1
);

SELECT 
    ac.constraint_name,
    ac.table_name AS child_table,
    acc.column_name AS child_column,
    ac_r.table_name AS parent_table,
    acc_r.column_name AS parent_column
FROM all_constraints ac
JOIN all_cons_columns acc 
    ON ac.constraint_name = acc.constraint_name
JOIN all_constraints ac_r 
    ON ac.r_constraint_name = ac_r.constraint_name
JOIN all_cons_columns acc_r
    ON ac_r.constraint_name = acc_r.constraint_name
WHERE ac.constraint_type = 'R'
  AND ac_r.table_name = 'TEST'
ORDER BY ac.table_name, ac.constraint_name;

-- 1. Identify duplicate test IDs (excluding the oldest)
WITH dup AS (
    SELECT id,
           ROW_NUMBER() OVER (
               PARTITION BY method_name, display_name, test_class_id
               ORDER BY created ASC
           ) AS rn
    FROM test
),
to_delete AS (
    SELECT id
    FROM dup
    WHERE rn > 1
)

-- 2. Delete child rows in correct order
DELETE FROM test_param
WHERE test_id IN (SELECT id FROM to_delete);

DELETE FROM test_run
WHERE test_id IN (SELECT id FROM to_delete);

DELETE FROM test_step
WHERE test_id IN (SELECT id FROM to_delete);

DELETE FROM test_tag
WHERE test_id IN (SELECT id FROM to_delete);

-- 3. Delete duplicate test rows
DELETE FROM test
WHERE id IN (SELECT id FROM to_delete);


```
