# MongoDB — Change Streams & Resume Tokens

---

## How It Works

```mermaid
flowchart LR
    subgraph MG["MongoDB"]
        OP[Oplog\nreplica set log]
        CS[Change Stream\ncursor]
        RT[Resume Token\nper event]
    end

    subgraph DBZ["Debezium"]
        WATCH[watch() cursor\nopen change stream]
        PARSE[Event parser]
        OFF[Offset: resume token\npersisted in Kafka]
    end

    OP --> CS
    CS -->|resume token| RT
    RT -.->|stored offset| OFF
    CS --> WATCH --> PARSE
```

Debezium opens a MongoDB Change Stream using `watch()` and processes events as they arrive. The **Resume Token** from each event is persisted as the offset — allowing the connector to resume from exactly where it left off after a restart.

---

## Prerequisites

```javascript
// MongoDB must be running as a replica set (required for change streams)
rs.initiate({
  _id: "rs0",
  members: [{ _id: 0, host: "mongodb:27017" }]
})

// Create Debezium user
use admin
db.createUser({
  user: "debezium",
  pwd: "SecurePassword123!",
  roles: [
    { role: "read", db: "mydb" },
    { role: "read", db: "local" },    // oplog access
    { role: "readAnyDatabase", db: "admin" }
  ]
})
```

---

## Connector Configuration

```json
{
  "name": "debezium-mongodb-prod",
  "config": {
    "connector.class": "io.debezium.connector.mongodb.MongoDbConnector",
    "mongodb.connection.string": "mongodb://debezium:password@mongodb:27017/?replicaSet=rs0",
    "topic.prefix": "prod-mongodb",
    "database.include.list": "mydb",
    "collection.include.list": "mydb.orders,mydb.customers",
    "snapshot.mode": "initial",
    "heartbeat.interval.ms": "10000",
    "capture.mode": "change_streams_update_full"
  }
}
```

---

## Resume Token — Critical Distinctions

### The `_id` Confusion

> ⚠️ In a MongoDB Change Stream event, `_id` is the **resume token of the event**, NOT the `_id` of the document.

```javascript
// Change Stream event structure
{
  "_id": { "_data": "8263F1E8..." },     // ← resume token (event identifier)
  "operationType": "update",
  "ns": { "db": "mydb", "coll": "orders" },
  "documentKey": { "_id": ObjectId("...") },  // ← actual document _id
  "fullDocument": { "_id": ObjectId("..."), "status": "shipped" }
}
```

Using `_id` as the deduplication key will use the resume token (correct for event identity), but the field naming is confusing and version-dependent.

### Correct Offset Key

```python
elif source_type == 'mongodb':
    # Debezium >= 2.0: resume_token exposed in source
    resume_token = F.col('source.resume_token')
    # Fallback: ts_ms + ord (ordinal within same timestamp)
    fallback = F.concat(
        F.col('source.ts_ms').cast('string'),
        F.lit('_'),
        F.coalesce(F.col('source.ord').cast('string'), F.lit('0'))
    )
    return df.withColumn('_offset_key', F.coalesce(resume_token, fallback))
```

> [!IMPORTANT]
> **Priority**: Always prioritize `resume_token` for deduplication. The `ts_ms + ord` fallback is less reliable.

> [!CAUTION]
> **Clock Skew in Sharded Clusters**: In MongoDB sharded clusters, `ts_ms` is derived from `clusterTime`. Due to potential clock skew between shards, `ts_ms` is NOT guaranteed to be globally monotonic. Use `resume_token` as the only source of truth for ordering.

---

## Cluster Migration — Critical Warning

> ⚠️ The resume token is **scoped to the current cluster**. It is not portable across clusters.

**Scenarios that invalidate resume tokens:**
- Migration from Atlas to on-premise (or vice versa)
- Blue/green cluster migration
- Cross-region failover to a different cluster
- Major version upgrade with oplog format change

**Recovery procedure after cluster migration:**
```bash
# 1. Delete the connector
curl -X DELETE http://kafka-connect:8083/connectors/debezium-mongodb-prod

# 2. Delete the stored offset from connect-offsets topic
# (requires kafka-consumer-groups.sh or offset reset tool)

# 3. Recreate connector with initial snapshot
curl -X POST http://kafka-connect:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{"config": {"snapshot.mode": "initial", ...}}'
```

For zero-loss SLA, test the migration procedure in staging **before** executing in production.

---

## Capture Modes

| Mode | Description |
|---|---|
| `change_streams` | Default — no `fullDocument` on UPDATE |
| `change_streams_update_full` | **Recommended** — fetches full document on UPDATE (extra lookup) |
| `change_streams_with_pre_image` | Requires MongoDB 6.0+ — provides `before` state natively |

For Silver layer MERGE pipelines, use `change_streams_update_full` or `change_streams_with_pre_image` (6.0+) to get the full document state after every UPDATE.
