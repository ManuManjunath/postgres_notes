# postgres_notes

To create a new postgres instance using Docker - 
```
docker run --name my-postgres -e POSTGRES_PASSWORD=secret -e POSTGRES_DB=<some_db_name> -p 5432:5432 -d postgres:13.3-alpine
```

Check all postgres processes - ``` ps auxww | grep ^postgres ```

Check all old idle sessions - 
```
select pid, usename, datname, application_name, age(now(),query_start), substr(query, 0, 75), wait_event_type, wait_event
from pg_stat_activity where datname = 'my_db' and state = 'idle' order by 5 desc;
```

Check grants - 
```
select * from information_schema.role_usage_grants where object_type = 'SEQUENCE';
```

Kill all idle in transaction - 
```
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity
WHERE datname = 'my_db'
AND pid <> pg_backend_pid()
AND state in ('idle in transaction');
```

Kill idle sessions - 
```
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity 
WHERE datname = 'my_db' 
AND pid <> pg_backend_pid() 
AND state in ('idle');
```

Conf file - ```/base_path/data/postgresql.conf```

Dead/Live tuples - 
```
SELECT relname AS TableName,
n_live_tup AS LiveTuples,
n_dead_tup AS DeadTuples
FROM pg_stat_user_tables;
```

Check connections - 
```
select datname,usename,client_addr,state,count(1) from pg_stat_activity group by datname,usename,client_addr,state ;
```

Blocking sessions - 

```
SELECT blocked_locks.pid     AS blocked_pid,
 		 blocked_activity.usename  AS blocked_user,
 		 blocking_locks.pid     AS blocking_pid,
 		 blocking_activity.usename AS blocking_user,
		 substr(blocked_activity.query,1,50)    AS blocked_statement,
		 substr(blocking_activity.query,1,50)   AS current_statement_in_blocking_process
 	   FROM  pg_catalog.pg_locks         blocked_locks
 	    JOIN pg_catalog.pg_stat_activity blocked_activity  ON blocked_activity.pid = blocked_locks.pid
 	    JOIN pg_catalog.pg_locks         blocking_locks
 		ON blocking_locks.locktype = blocked_locks.locktype
 		AND blocking_locks.DATABASE IS NOT DISTINCT FROM blocked_locks.DATABASE
 		AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
 		AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
 		AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
 		AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
 		AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
 		AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
 		AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
 		AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
 		AND blocking_locks.pid != blocked_locks.pid
 	    JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
 	   WHERE NOT blocked_locks.GRANTED;

select pg_blocking_pids(pid) as blockers,
age(now(),
query_start), 
pid, 
state, 
substr(query,0,50) 
from pg_stat_activity 
where state != 'idle' 
order by 2 desc	   
```


Check dependency -

```
SELECT v.oid::regclass AS view
FROM pg_depend AS d      -- objects that depend on the table
   JOIN pg_rewrite AS r  -- rules depending on the table
      ON r.oid = d.objid
   JOIN pg_class AS v    -- views for the rules
      ON v.oid = r.ev_class
WHERE v.relkind = 'v'    -- only interested in views
  -- dependency must be a rule depending on a relation
  AND d.classid = 'pg_rewrite'::regclass
  AND d.refclassid = 'pg_class'::regclass
  AND d.deptype = 'n'    -- normal dependency
  AND d.refobjid = 'scott1'::regclass;
```

Table usage - 

```
select schemaname || '.' || relname as table_name,
seq_scan as table_reads,
idx_scan as idx_reads,
n_tup_ins as inserts, 
n_tup_upd as updates, 
n_tup_del as deletes 
from pg_stat_user_tables order by 2 ;
```


```
     SELECT
       pid,
       now() - pg_stat_activity.query_start AS duration,
       query,
       state
     FROM pg_stat_activity
     WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes';
```

View definition -
```
select pg_get_viewdef('view_name', true);
```

Roles and permissions - 
```
SELECT r.rolname, r.rolsuper, r.rolinherit,
  r.rolcreaterole, r.rolcreatedb, r.rolcanlogin,
  r.rolconnlimit, r.rolvaliduntil,
  ARRAY(SELECT b.rolname
        FROM pg_catalog.pg_auth_members m
        JOIN pg_catalog.pg_roles b ON (m.roleid = b.oid)
        WHERE m.member = r.oid) as memberof
, r.rolreplication
, r.rolbypassrls
FROM pg_catalog.pg_roles r
WHERE r.rolname !~ '^pg_'
ORDER BY 1;
```

To copy an entire Database - 
```
pg_dump --host <hostname> --port 5432 --username <user> --verbose --file "path/to/file.sql" --dbname <db_name>
```

By Table -
```
pg_dump -g <hostname> --username <user> --dbname <db_name> --verbose --table <tbl_name> --file "</path/to/file.sql>"
```

To copy DDL only (especially useful for compliance servers - 
```
/usr/pgsql-13/bin/pg_dump -U postgres -h localhost -t '<schema_name>.<>db_name' --schema-only <db_name> > <file_namew>.ddl
```

To restore:
```
psql -h <hostname> -U <user> <db_name> < "/path/to/file.sql"
```

List all sequences in a DB
```
SELECT c.relname FROM pg_class c WHERE c.relkind = 'S';
```

Grants:
```
Grant usage,select on <sequences_name> to pgm_<DatabaseName>_ro
Grant usage,select on <sequences_name> to pgm_<DatabaseName>_rw
```

Transactions
```
BEGIN:
    UPDATE TABLE
SAVEPOINT <name>; --> PG remembers this point in time.
> Make some incorrect updates.
ROLLBACK TO <name>; --> Rolls back to savepoint
> Make the right updates.
COMMIT;
```

Check existing grants on sequences -
```
SELECT relname, relacl
FROM pg_class
WHERE relkind = 'S'
  AND relacl is not null
  AND relnamespace IN (
      SELECT oid
      FROM pg_namespace
      WHERE nspname NOT LIKE 'pg_%'
        AND nspname != 'information_schema'
);
```

To set default schema: ```set search_path = 'public';```


