---
title: "PostgreSQL Database Migration with Zero Downtime"
date: "2025-03-12"
author:
  name: Charles
  picture: "/assets/blog/authors/cc.png"
categories: 
  - "instruction"
coverImage: "/assets/blog/hello-world/cover.jpg"
ogImage:
  url: "/assets/blog/preview/cover.jpg"
---

# PostgreSQL Database Migration with Zero Downtime

## Standard Operating Procedure (SOP)

## 1. Introduction

This document outlines the procedure for migrating a PostgreSQL database (version 17) from a local environment to the cloud with minimal or zero downtime using logical replication. This method allows the source database to continue operating while data is synchronized to the cloud instance.

## 2. Prerequisites

- PostgreSQL 17 running locally
- A cloud-based PostgreSQL instance (AWS, GCP, Azure, or other)
- Network connectivity between the source and target databases
- Administrative access to both database instances
- Sufficient disk space on both instances

## 3. Pre-Migration Tasks

### 3.1 Verify Database Compatibility

Ensure that the source and target PostgreSQL versions are compatible for replication. Ideally, the target version should be the same as or higher than the source version.

### 3.2 Assess Database Size and Structure

```sql
-- Check database size
SELECT pg_size_pretty(pg_database_size('dbname'));

-- Get table list and sizes
SELECT
    table_schema,
    table_name,
    pg_size_pretty(pg_total_relation_size('"' || table_schema || '"."' || table_name || '"')) as size
FROM
    information_schema.tables
WHERE
    table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY
    pg_total_relation_size('"' || table_schema || '"."' || table_name || '"') DESC;
```

### 3.3 Identify Large Tables

Large tables might require special attention during migration to minimize replication lag.

### 3.4 Check Primary Keys

Logical replication requires primary keys or replica identity. Verify all tables have appropriate identifiers:

```sql
SELECT 
    t.table_schema, 
    t.table_name
FROM 
    information_schema.tables t
LEFT JOIN 
    information_schema.table_constraints c 
    ON c.table_schema = t.table_schema 
    AND c.table_name = t.table_name 
    AND c.constraint_type = 'PRIMARY KEY'
WHERE 
    t.table_schema NOT IN ('pg_catalog', 'information_schema')
    AND t.table_type = 'BASE TABLE'
    AND c.constraint_name IS NULL;
```

## 4. Configure Source Database

### 4.1 Enable Logical Replication

Edit the `postgresql.conf` file to enable logical replication:

```
wal_level = logical
max_replication_slots = 10  # Adjust based on needs
max_wal_senders = 10        # Adjust based on needs
max_logical_replication_workers = 4
max_worker_processes = 10
```

### 4.2 Restart PostgreSQL

This step requires a brief downtime. Schedule it during a low-traffic period:

```bash
# For systemd-based systems:
sudo systemctl restart postgresql

# For init.d-based systems:
sudo service postgresql restart

# For manual installations:
pg_ctl restart -D /path/to/data/directory
```

### 4.3 Configure Network Access

Edit `pg_hba.conf` to allow replication connections from the target database:

```
# Add the following line, adjusting for your target IP
host    replication     replication_user    target_ip/32     md5
```

### 4.4 Create a Replication User

```sql
-- Create a dedicated user for replication
CREATE ROLE replication_user WITH LOGIN PASSWORD 'secure_password' REPLICATION;

-- Grant necessary permissions
GRANT USAGE ON SCHEMA public TO replication_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replication_user;
```

### 4.5 Create Publication

For each database you want to replicate:

```sql
-- Connect to the appropriate database
\c dbname

-- Create publication for all tables
CREATE PUBLICATION my_publication FOR ALL TABLES;

-- Alternatively, for specific tables
-- CREATE PUBLICATION my_publication FOR TABLE table1, table2, table3;
```

### 4.6 Verify Publication

```sql
-- View all publications
SELECT * FROM pg_publication;

-- View detailed information
SELECT 
    p.pubname AS publication_name,
    p.puballtables AS publishes_all_tables,
    p.pubinsert AS replicates_inserts,
    p.pubupdate AS replicates_updates,
    p.pubdelete AS replicates_deletes,
    p.pubtruncate AS replicates_truncates,
    pt.schemaname AS schema_name,
    pt.tablename AS table_name
FROM 
    pg_publication p
LEFT JOIN 
    pg_publication_tables pt 
ON 
    p.pubname = pt.pubname
ORDER BY 
    p.pubname, pt.schemaname, pt.tablename;
```

## 5. Configure Target Database

### 5.1 Create Database Schema

Create the same database schema on the target as on the source:

```bash
# Option 1: Using pg_dump for schema only
pg_dump -h source_host -U source_user -d source_db --schema-only > schema.sql
psql -h target_host -U target_user -d target_db -f schema.sql

# Option 2: For multiple databases
for db in db1 db2 db3; do
  pg_dump -h source_host -U source_user -d $db --schema-only > ${db}_schema.sql
  psql -h target_host -U target_user -d $db -f ${db}_schema.sql
done
```

### 5.2 Enable Logical Replication on Target

Just like the source, the target database also needs logical replication enabled:

```
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

This requires a restart of the target PostgreSQL instance.

## 6. Set Up Replication

### 6.1 Create Subscription

For each database you want to replicate, connect to the corresponding database on the target and create a subscription:

```sql
-- Connect to the appropriate target database
\c target_dbname

CREATE SUBSCRIPTION my_subscription
CONNECTION 'host=source_host port=5432 dbname=source_dbname user=replication_user password=secure_password'
PUBLICATION my_publication;
```

Example for multiple databases:

```sql
-- For database 1
CREATE SUBSCRIPTION db1_subscription
CONNECTION 'host=192.168.1.43 port=5432 dbname=db1 user=replication_user password=secure_password'
PUBLICATION db1_publication;

-- For database 2
CREATE SUBSCRIPTION db2_subscription
CONNECTION 'host=192.168.1.43 port=5432 dbname=db2 user=replication_user password=secure_password'
PUBLICATION db2_publication;
```

### 6.2 Monitor Initial Synchronization

The initial synchronization process can take time depending on the size of your database. Monitor its progress:

```sql
-- Check subscription status
SELECT * FROM pg_stat_subscription;

-- For more detailed information:
SELECT 
    subname, 
    subenabled, 
    pg_size_pretty(pg_subscription_rel_size(subname::regclass)) as subscription_size, 
    srsubstate, 
    srrelid::regclass 
FROM 
    pg_subscription, 
    pg_subscription_rel
WHERE 
    pg_subscription.oid = pg_subscription_rel.srsubid
ORDER BY 
    pg_subscription_rel_size(subname::regclass) DESC;
```

## 7. Test and Verify Replication

### 7.1 Test Data Changes

Make test changes on the source database and verify they appear on the target:

```sql
-- On source (create a test record)
INSERT INTO test_table (id, name) VALUES (999, 'Replication Test');

-- On target (verify the record exists)
SELECT * FROM test_table WHERE id = 999;
```

### 7.2 Verify Replication Lag

Monitor replication lag to ensure data is being synchronized promptly:

```sql
SELECT 
    now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

## 8. Handle Schema Changes

Schema changes require special handling during logical replication:

### 8.1 Adding a Column

```sql
-- 1. First, add the column to the target database
-- Connect to target database
ALTER TABLE users ADD COLUMN email_verified boolean DEFAULT false;

-- 2. Then add the same column to the source database
-- Connect to source database  
ALTER TABLE users ADD COLUMN email_verified boolean DEFAULT false;
```

### 8.2 Adding a New Table

```sql
-- 1. Create the table on both source and target databases

-- 2. Add the table to the publication on source
ALTER PUBLICATION my_publication ADD TABLE new_table;

-- 3. Refresh the subscription on target
ALTER SUBSCRIPTION my_subscription REFRESH PUBLICATION;
```

## 9. Cutover Procedure

### 9.1 Pre-Cutover Checklist

- Confirm replication is working correctly with minimal lag
- Verify all schema changes have been synchronized
- Test the target database with a test application instance
- Schedule a maintenance window if possible

### 9.2 Cutover Steps

1. Stop all write operations to the source database
2. Verify that replication is caught up (replication_lag should be near zero)
3. Update connection strings in all applications to point to the new database
4. Start applications with the new connection strings
5. Monitor the new database for any issues

### 9.3 Post-Cutover

- Keep the source database running for a few days as a backup
- Monitor performance on the new cloud database
- Adjust cloud database parameters if needed for optimal performance

## 10. Clean Up

Once you're confident in the migration, clean up the replication components:

```sql
-- On the target database
DROP SUBSCRIPTION my_subscription;

-- On the source database
DROP PUBLICATION my_publication;
```

## 11. Troubleshooting

### 11.1 Common Issues and Solutions

#### Subscription Creation Fails
```
ERROR: could not create replication slot "subscription_name": ERROR: logical decoding requires "wal_level" >= "logical"
```
**Solution**: Ensure `wal_level = logical` is set on both source and target databases and both have been restarted.

#### Tables Missing from Replication
**Solution**: Check if tables have primary keys or replica identity set:
```sql
ALTER TABLE table_name REPLICA IDENTITY FULL;
```

#### Replication Lag Growing
**Solution**: Check for:
- Network bandwidth limitations
- Disk I/O bottlenecks
- High write load on source database
- Consider increasing `max_logical_replication_workers`

## 12. Additional Considerations

### 12.1 Large Tables Strategy

For very large tables, consider:
- Initial data copy using `pg_dump/pg_restore`
- Creating the subscription after the initial load
- Using table partitioning for improved performance

### 12.2 Monitoring

Set up continuous monitoring for:
- Replication lag
- Replication slot size
- WAL generation rate
- CPU/Memory/Disk usage on both servers

---

## Appendix A: Quick Reference Commands

### View Publications
```sql
SELECT * FROM pg_publication;
```

### View Subscriptions
```sql
SELECT * FROM pg_subscription;
```

### Check Replication Status
```sql
SELECT * FROM pg_stat_subscription;
```

### Check Replication Lag
```sql
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

### Pause Replication
```sql
ALTER SUBSCRIPTION my_subscription DISABLE;
```

### Resume Replication
```sql
ALTER SUBSCRIPTION my_subscription ENABLE;
```

---

Document Version: 1.0
Last Updated: March 12, 2025
