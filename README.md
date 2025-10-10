```sql

INSERT INTO dashboard (
    id,
    created,
    last_updated,
    name,
    description,
    is_default,
    deleted,
    user_id
)
VALUES (
    dashboard_seq.NEXTVAL,        -- auto-generate ID using your Oracle sequence
    SYSTIMESTAMP,                 -- creation timestamp
    SYSTIMESTAMP,                 -- last updated timestamp
    'Global System Dashboard',    -- name of the system dashboard
    'Default dashboard available to all users',  -- description
    1,                            -- is_default = true (system dashboard)
    0,                            -- deleted = false
    NULL                          -- user_id = NULL (belongs to the system)
);

```
