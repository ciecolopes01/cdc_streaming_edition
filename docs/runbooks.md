# Runbooks — Incident Response

Production-ready runbooks for the most common CDC incidents. Each runbook follows the same structure: Symptoms → Diagnose → Remediate → Prevent.

---

## RB-001: PostgreSQL Replication Slot — WAL Lag > 50GB

**Symptoms**
- PostgreSQL disk growing rapidly (no data volume growth)
- Autovacuum showing no progress
- Alert: `wal_lag_bytes > 10737418240` (10GB)
- Alert: `age(datfrozenxid) > 1000000000`

**Diagnose**

```sql
-- Step 1: Confirm the problem and identify the slot
SELECT
    slot_name,
    active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag,
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)                 AS lag_bytes
FROM pg_replication_slots
ORDER BY lag_bytes DESC;

-- Step 2: Check wraparound risk
SELECT datname, age(datfrozenxid) AS xid_age
FROM pg_database
ORDER BY xid_age DESC;
```

```bash
# Step 3: Check connector health
curl -s http://kafka-connect:8083/connectors/debezium-prod-pg/status | jq .
```

**Remediate**

```bash
# Option A: Connector RUNNING but slow → increase resources, monitor
# Check Kafka Connect worker CPU/memory and Kafka consumer lag

# Option B: Connector task FAILED → restart task
curl -X POST http://kafka-connect:8083/connectors/debezium-prod-pg/tasks/0/restart
sleep 60
curl -s http://kafka-connect:8083/connectors/debezium-prod-pg/status | jq '.tasks[].state'

# Option C: Not recovering after 30 min → drop slot and re-snapshot
# DECISION REQUIRED: dropping slot means new snapshot (hours of data catch-up)
psql -c "SELECT pg_drop_replication_slot('debezium_prod');"

# Option D: Wraparound imminent → emergency vacuum FIRST, then deal with connector
psql -c "VACUUM (FREEZE, VERBOSE, ANALYZE) public.most_active_table;"
```

```bash
# Recreate connector after slot drop
curl -X DELETE http://kafka-connect:8083/connectors/debezium-prod-pg
curl -X POST http://kafka-connect:8083/connectors \
  -H "Content-Type: application/json" \
  -d @examples/postgres-connector.json
```

**Prevent**

```ini
# postgresql.conf
max_slot_wal_keep_size = 20GB   # auto-invalidate slot before disk fills
```

```sql
-- Alert rule: slot inactive
SELECT slot_name FROM pg_replication_slots WHERE NOT active;

-- Alert rule: lag > 10GB
SELECT slot_name
FROM pg_replication_slots
WHERE pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) > 10737418240;
```

---

## RB-002: Task FAILED — Schema Error

**Symptoms**
- Task in FAILED state
- Error in trace: `Schema not found`, `Incompatible schema`, or `SerializationException`
- Consumers may be receiving no new events

**Diagnose**

```bash
# Step 1: Read the full error trace
curl -s http://kafka-connect:8083/connectors/debezium-prod-pg/status \
  | jq '.tasks[] | select(.state == "FAILED") | .trace'

# Step 2: Check Schema Registry for the affected subject
curl -s http://schema-registry:8081/subjects | jq .
curl -s http://schema-registry:8081/subjects/prod-pg.public.orders-value/versions | jq .

# Step 3: Test compatibility before registering new schema
curl -X POST http://schema-registry:8081/compatibility/subjects/prod-pg.public.orders-value/versions/latest \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{"schema": "<avro_schema_json>"}'
```

**Remediate**

```bash
# Option A: Schema incompatibility → temporarily set NONE, then fix properly
curl -X PUT http://schema-registry:8081/config/prod-pg.public.orders-value \
  -H "Content-Type: application/json" \
  -d '{"compatibility": "NONE"}'

# Restart task after schema is resolved
curl -X POST http://kafka-connect:8083/connectors/debezium-prod-pg/tasks/0/restart

# IMPORTANT: revert compatibility mode after fixing
curl -X PUT http://schema-registry:8081/config/prod-pg.public.orders-value \
  -H "Content-Type: application/json" \
  -d '{"compatibility": "BACKWARD_TRANSITIVE"}'

# Option B: Check DLQ for events that failed during the outage
kafka-console-consumer.sh --bootstrap-server kafka:9092 \
  --topic dlq.debezium.prod-pg \
  --from-beginning \
  --max-messages 500 \
  --property print.headers=true
```

**Prevent**

- Set all subjects to `BACKWARD_TRANSITIVE` (not just `BACKWARD`)
- Enforce DDL review process for all tables captured by Debezium
- Test schema changes in staging with the same Registry compatibility settings

---

## RB-003: Oracle — ORA-01291 Missing Logfile

**Symptoms**
- Connector in FAILED state with `ORA-01291: missing logfile` in trace
- Oracle archive logs were deleted by RMAN before Debezium read them

**Diagnose**

```bash
# Step 1: Get the SCN Debezium is trying to read
curl -s http://kafka-connect:8083/connectors/debezium-oracle-prod/offsets | jq .
# Note the scn value
```

```sql
-- Step 2: Check if archive log for that SCN still exists
SELECT name, first_change#, next_change#, deleted
FROM v$archived_log
WHERE first_change# <= :debezium_scn
  AND next_change#  > :debezium_scn;

-- Step 3: Check current RMAN retention policy
SELECT 'CONFIGURE ARCHIVELOG RETENTION POLICY TO RECOVERY WINDOW OF ' ||
       value || ' DAYS;' AS current_policy
FROM v$rman_configuration
WHERE name = 'ARCHIVELOG RETENTION POLICY';
```

**Remediate**

```bash
# Option A: Archive log exists but marked deleted → restore via RMAN
rman target / <<EOF
RESTORE ARCHIVELOG FROM SCN <debezium_scn>;
EOF

# Restart connector after restore
curl -X POST http://kafka-connect:8083/connectors/debezium-oracle-prod/tasks/0/restart
```

```bash
# Option B: Archive log is gone → must re-snapshot
curl -X DELETE http://kafka-connect:8083/connectors/debezium-oracle-prod

# Recreate with initial snapshot
curl -X POST http://kafka-connect:8083/connectors \
  -H "Content-Type: application/json" \
  -d @examples/oracle-connector.json
```

**Prevent**

```bash
# Set RMAN retention to at least 7 days (or 2× connector downtime SLA)
rman target / <<EOF
CONFIGURE ARCHIVELOG RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
EOF
```

---

## RB-004: MongoDB — InvalidResumeToken after Cluster Migration

**Symptoms**
- Connector in FAILED state with `InvalidResumeToken` error
- Occurred after a MongoDB cluster migration (Atlas → on-prem, version upgrade, etc.)

**Diagnose**

```bash
# Confirm error
curl -s http://kafka-connect:8083/connectors/debezium-mongodb-prod/status \
  | jq '.tasks[] | select(.state == "FAILED") | .trace'
# Look for: InvalidResumeToken or ChangeStreamHistoryLost
```

**Remediate**

```bash
# Step 1: Delete connector (this stops the failing restart loop)
curl -X DELETE http://kafka-connect:8083/connectors/debezium-mongodb-prod

# Step 2: Clear stored offset using the REST API (Kafka Connect 3.5+)
# Source connector offsets are in connect-offsets, NOT in consumer groups
curl -X DELETE http://kafka-connect:8083/connectors/debezium-mongodb-prod/offsets

# Step 3: Recreate with full initial snapshot
curl -X POST http://kafka-connect:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "debezium-mongodb-prod",
    "config": {
      "snapshot.mode": "initial",
      ...
    }
  }'
```

**Prevent**

- Document in any cluster migration runbook: **resume tokens are cluster-scoped**
- Test migration in staging, verify connector resumes, before running in production
- Keep `snapshot.mode=initial` available as a fallback in your connector config template

---

## RB-005: Tombstone Lost — Ghost Data in Silver Layer

**Symptoms**
- Records deleted in the source database still appear in Silver layer
- DML audit shows DELETE events in Bronze but records not removed from Silver
- Occurred after consumer was offline for extended period

**Diagnose**

```bash
# Check topic configuration
kafka-topics.sh --bootstrap-server kafka:9092 \
  --describe --topic prod-pg.public.orders

# Look for:
# cleanup.policy = compact,delete  ← dangerous
# retention.ms   = <low value>     ← risky
# delete.retention.ms = 86400000   ← 24h default — may be too low
```

**Remediate**

```bash
# 1. Identify affected records (compare source vs Silver)
# Run reconciliation query between source DB and Silver layer

# 2. Manually apply deletes in Silver
# Delta Lake
spark.sql("""
    MERGE INTO silver.orders AS target
    USING (SELECT id FROM source_db.deleted_ids) AS source
    ON target.id = source.id
    WHEN MATCHED THEN DELETE
""")

# 3. Fix topic configuration going forward
kafka-configs.sh --bootstrap-server kafka:9092 \
  --alter --entity-type topics --entity-name prod-pg.public.orders \
  --add-config 'cleanup.policy=compact,delete.retention.ms=172800000'
# Change compact,delete → compact if historical DELETE coverage is critical
```

**Prevent**

```properties
# Topic config: use compact only (not compact,delete) for tables with DELETE semantics
cleanup.policy=compact
delete.retention.ms=172800000   # 48h — must exceed consumer max downtime SLA
```

---

## Post-Mortems

### PM-001 — Slot Inactive → Wraparound (47min write outage)

Connector fell silently after credential rotation. Slot remained but inactive for 72h. WAL grew to 180GB. `xid_age` reached 1.8B. PostgreSQL entered wraparound protection mode.

**Root cause**: no alert for inactive slots + no `max_slot_wal_keep_size`.  
**Resolution**: force-drop slot, emergency `VACUUM FREEZE`, re-snapshot with incremental snapshot.  
**Lessons**: `max_slot_wal_keep_size=30GB`, alert on `active=false` for any slot.

---

### PM-002 — DDL Without Review Broke 23 Consumers

DBA added `NOT NULL` column without default on high-traffic table. Debezium published incompatible schema. Consumers with `BACKWARD` mode failed on deserialization.

**Root cause**: no DDL review process + `BACKWARD` instead of `BACKWARD_TRANSITIVE`.  
**Resolution**: revert DDL, migrate consumers, re-apply as nullable first.  
**Lessons**: mandatory DDL review for all CDC-captured tables. Schema Registry set to `BACKWARD_TRANSITIVE`.

---

### PM-003 — Tombstone Removed → Ghost Data in Silver

Topic configured as `compact,delete` with `retention.ms=86400000` (24h). Consumer offline for 36h. Tombstones for deleted records vanished. Silver retained stale rows.

**Root cause**: `compact,delete` with `retention.ms` shorter than consumer downtime SLA.  
**Resolution**: full Silver reprocess from Bronze. 6 hours of manual work.  
**Lessons**: Use `compact` only. Set `delete.retention.ms` ≥ 2× max downtime SLA.

---

### PM-004 — MongoDB Resume Token Invalid after Cluster Migration

Atlas → on-premise migration. Old resume token persisted in Kafka Connect. Connector looped on `InvalidResumeToken`.

**Root cause**: resume token undocumented as cluster-scoped. No migration runbook.  
**Resolution**: delete connector, clear offset, re-snapshot.  
**Lessons**: add resume token scope warning to all cluster migration runbooks.

---

### PM-005 — Oracle: Archive Log Deleted Before Debezium Read (8h re-snapshot)

RMAN retention set to 1 day. Connector down for 26h maintenance. Archive logs rotated. `ORA-01291`. RMAN backup also rotated — no restore possible. Re-snapshot of 200M records took 8h.

**Root cause**: RMAN retention shorter than connector maintenance window.  
**Resolution**: forced full re-snapshot.  
**Lessons**: RMAN retention ≥ 7 days. Alert when `seconds since last archive > threshold`.

---

### PM-006 — Oracle RAC: Redo Log Inaccessible after ASM Mount Failure

Node 2 redo logs went to local storage after ASM remount failure post-maintenance. LogMiner could not access them — `ORA-00308`. Connector stalled.

**Root cause**: ASM not validated after node maintenance + no automated redo log accessibility check.  
**Resolution**: manual ASM remount on node 2. Connector resumed automatically.  
**Lessons**: ASM mount status on all RAC nodes must be validated in maintenance checklist.
