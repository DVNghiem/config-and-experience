## Detailed PostgreSQL Configuration for High-Traffic Systems & Best Practices

### Core Configuration (postgresql.conf)
```ini
#-----------------------------
# Memory Settings
#-----------------------------
shared_buffers = 16GB                # 25-40% of total RAM (e.g., 64GB RAM → 16-25GB)
work_mem = 64MB                     # Memory per operation (sorts, hash joins)
maintenance_work_mem = 2GB          # For VACUUM, REINDEX, CREATE INDEX
effective_cache_size = 48GB         # 75% of total RAM (e.g., 64GB → 48GB)
huge_pages = try                    # Enable if supported by OS

#-----------------------------
# Connections & Replication
#-----------------------------
max_connections = 500               # Use PgBouncer for connection pooling
superuser_reserved_connections = 5
listen_addresses = '*'              # Allow remote connections (restrict via firewall)
wal_level = replica                 # Required for replication
max_wal_senders = 10                # Number of replication slots

#-----------------------------
# WAL & Checkpoint
#-----------------------------
wal_buffers = 16MB                  # 3% of shared_buffers (max 16MB)
checkpoint_timeout = 15min          # Time between checkpoints
max_wal_size = 32GB                 # Reduce checkpoint frequency
min_wal_size = 8GB
checkpoint_completion_target = 0.9  # Spread I/O during checkpoints
wal_compression = on                # Compress WAL (PostgreSQL 13+)

#-----------------------------
# Query Optimization
#-----------------------------
random_page_cost = 1.1              # Lower for SSDs (default: 4.0 for HDDs)
effective_io_concurrency = 200      # Optimize for SSDs/NVMe
max_parallel_workers_per_gather = 8 # Parallel workers per query
max_parallel_workers = 16          # Total parallel workers
parallel_leader_participation = on  # Allow leader to participate in processing
jit = off                          # Disable JIT unless needed for complex queries

#-----------------------------
# Autovacuum
#-----------------------------
autovacuum = on
autovacuum_max_workers = 6          # Number of autovacuum processes
autovacuum_vacuum_cost_limit = 2000 # Speed up autovacuum
autovacuum_vacuum_scale_factor = 0.1 # Trigger autovacuum earlier
log_autovacuum_min_duration = 0     # Log all autovacuum activity

#-----------------------------
# Logging & Monitoring
#-----------------------------
logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d.log'
log_min_duration_statement = 5000   # Log queries slower than 5 seconds
shared_preload_libraries = 'pg_stat_statements, pg_prewarm'
pg_stat_statements.track = all      # Track all queries for analysis
```

### Best Practices for High Traffic
1. Connection Pooling (PgBouncer)
    - Use transaction pooling to manage connections efficiently.
    ```ini
    [pgbouncer]
    pool_mode = transaction
    max_client_conn = 10000
    default_pool_size = 50
    reserve_pool_size = 10
    ```
2. Replication & High Availability
    - Deploy streaming replication (Primary → Standby) for fault tolerance.
    - Use tools like Patroni or repmgr for automated failover.
3. Table Partitioning & Sharding
    - Partition large tables by time or range:
    ```sql
    CREATE TABLE logs (log_date DATE) PARTITION BY RANGE (log_date);
    ```
    - Use `Citus` for horizontal sharding:
    ```sql
    SELECT create_distributed_table('orders', 'user_id');
    ```
4. Index Optimization
    - Create indexes on frequently queried columns.
    - Avoid over-indexing to reduce write overhead.
    ```sql
    CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
    ```
### Monitoring & Maintenance
1. Query Analysis
    - Identify slow queries with pg_stat_statements:
    ```sql
    SELECT query, total_time, calls FROM pg_stat_statements ORDER BY total_time DESC LIMIT 10;
    ```
2. Vacuum & Analyze
    - Schedule regular VACUUM and ANALYZE jobs:
    ```sql
    vacuumdb --analyze --verbose --dbname=mydb
    ```
3. Prometheus + Grafana
    - Monitor key metrics:
    - TPS (Transactions Per Second)
    - Buffer cache hit ratio
    - Replication lag
    - Lock contention
### Hardware & OS Tuning
1. Hardware Recommendations:
    - RAM: 64GB+ (for OLTP workloads).
    - Storage: NVMe SSDs (RAID 10 for redundancy).
    - CPU: 16+ cores for parallel processing.

2. OS/Kernel Tuning:
```ini
# /etc/sysctl.conf
kernel.shmmax = 17179869184     # 16GB (must ≥ shared_buffers)
vm.swappiness = 1               # Minimize swapping
vm.dirty_background_ratio = 5   # Tune writeback behavior
vm.dirty_ratio = 10
```
### Backup & Disaster Recovery
1. Physical Backups
```bash
pg_basebackup -D /backup/path -X stream -P -U replicator
```
2. WAL Archiving
```ini
# postgresql.conf
archive_mode = on
archive_command = 'gzip < %p > /wal_archive/%f.gz'
```
3. Point-in-Time Recovery (PITR)
    - Restore from a base backup and replay WAL logs.

### Security
1. SSL Encryption
```ini
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file = '/etc/ssl/private/server.key'
```
2. Role-Based Access Control (RBAC)
```sql
CREATE ROLE analyst;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO analyst;
```

### Critical Notes
- Load Testing: Use pgbench to simulate traffic:
```bash
pgbench -c 200 -j 8 -T 600 mydb
```
- Avoid Long Transactions: Commit transactions quickly to reduce locks.
- Index Wisely: Over-indexing degrades write performance.
- Regular Maintenance: Monitor bloat and tune autovacuum thresholds.
