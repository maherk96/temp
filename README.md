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


```
