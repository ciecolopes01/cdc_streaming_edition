# CDC Enterprise Extreme

> **Production-grade Change Data Capture with Debezium â€” The Definitive Guide**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](CONTRIBUTING.md)
[![Stars](https://img.shields.io/github/stars/your-org/cdc-enterprise-extreme?style=social)](https://github.com/your-org/cdc-enterprise-extreme)

A community-driven, battle-tested reference for running **CDC pipelines at enterprise scale**. Built from real production incidents, iterative technical corrections, and deep peer review.

Covers **PostgreSQL, MySQL, SQL Server, MongoDB, and Oracle** â€” with Debezium, Kafka, Schema Registry, Delta Lake, and Spark.

---

## ðŸ“– Documentation

| Section | Description |
|---|---|
| [Architecture](docs/architecture.md) | End-to-end pipeline design, component roles, data flow |
| [PostgreSQL](docs/postgresql.md) | Replication slots, WAL, vacuum, wraparound |
| [MySQL](docs/mysql.md) | Binlog, GTID, failover offset strategy |
| [SQL Server](docs/sqlserver.md) | change_lsn vs commit_lsn, CT vs CDC |
| [MongoDB](docs/mongodb.md) | Resume tokens, change streams, cluster migration |
| [Oracle](docs/oracle.md) | LogMiner, redo logs, supplemental logging, RAC |
| [Operations](docs/operations.md) | Monitoring, connector lifecycle, DLQ, rolling restarts |
| [Runbooks](docs/runbooks.md) | Incident response procedures â€” ready to use |
| [Performance](docs/performance.md) | Debezium tuning, Kafka producer/consumer, topic config |

---

## ðŸ—ºï¸ Architecture Overview

```mermaid
flowchart LR
    subgraph Sources["Source Databases"]
        PG[(PostgreSQL\nWAL / LSN)]
        MY[(MySQL\nBinlog / GTID)]
        SS[(SQL Server\nCDC / LSN)]
        MG[(MongoDB\nChange Streams)]
        OR[(Oracle\nLogMiner / SCN)]
    end

    subgraph Connect["Kafka Connect Cluster"]
        D1[Debezium\nConnector]
        D2[Debezium\nConnector]
        D3[Debezium\nConnector]
    end

    subgraph Kafka["Apache Kafka"]
        SR[Schema Registry\nAvro / BACKWARD_TRANSITIVE]
        T1[Topic: db.schema.table\ncleanup=compact]
        T2[Topic: schema-history\nretention=-1]
        DLQ[Dead Letter Queue\nretention=30d]
    end

    subgraph Processing["Stream Processing"]
        SP[Apache Spark\nStructured Streaming]
        DD[Deduplication\nby offset_key]
    end

    subgraph Lakehouse["Delta Lakehouse"]
        BZ[Bronze\nRaw CDC events\nSSE-KMS encrypted]
        SV[Silver\nDeduplicated\nMerged state]
        GD[Gold\nAggregated\nBusiness entities]
    end

    PG -->|WAL| D1
    MY -->|Binlog| D2
    OR -->|Redo Logs| D3
    SS -->|CT Log| D1
    MG -->|Oplog| D2

    D1 & D2 & D3 --> SR
    D1 & D2 & D3 --> T1
    D1 & D2 & D3 --> T2
    D1 & D2 & D3 -->|errors| DLQ

    T1 --> SP
    SP --> DD
    DD --> BZ --> SV --> GD
```

---

## ðŸš€ Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/your-org/cdc-enterprise-extreme.git
cd cdc-enterprise-extreme
```

### 2. Deploy a connector

```bash
# PostgreSQL
curl -X POST http://kafka-connect:8083/connectors \
  -H "Content-Type: application/json" \
  -d @examples/postgres-connector.json

# Oracle
curl -X POST http://kafka-connect:8083/connectors \
  -H "Content-Type: application/json" \
  -d @examples/oracle-connector.json

# MySQL
curl -X POST http://kafka-connect:8083/connectors \
  -H "Content-Type: application/json" \
  -d @examples/mysql-connector.json

# SQL Server
curl -X POST http://kafka-connect:8083/connectors \
  -H "Content-Type: application/json" \
  -d @examples/sqlserver-connector.json

# MongoDB
curl -X POST http://kafka-connect:8083/connectors \
  -H "Content-Type: application/json" \
  -d @examples/mongodb-connector.json
```

### 3. Verify connector health

```bash
curl -s http://kafka-connect:8083/connectors/debezium-prod-pg/status | jq .
```

---

## âš¡ Key Concepts at a Glance

### Offset Keys by Source

| Source | Offset Mechanism | Field | Notes |
|---|---|---|---|
| PostgreSQL | WAL LSN | `source.lsn` | Monotonic per instance |
| MySQL | Binlog position | `source.gtid` or `file:pos` | GTID preferred in HA setups |
| SQL Server | LSN | `source.change_lsn` | NOT `source.lsn` â€” silent null |
| MongoDB | Resume Token | `source.resume_token` | `ts_ms+ord` fallback only |
| Oracle | SCN | `commit_scn` + `scn` + `xid` | Include XID for uniqueness |

### Critical Production Rules

- âœ… Always use `cleanup.policy=compact` (not `compact,delete`) for CDC topics
- âœ… Set `delete.retention.ms` â‰¥ 2Ã— your consumer downtime SLA
- âœ… Never use `cleanup.policy=compact` on schema history topics â€” use `delete` with `retention.ms=-1`
- âœ… Use `BACKWARD_TRANSITIVE` schema compatibility, not just `BACKWARD`
- âœ… Set `max_slot_wal_keep_size` in PostgreSQL 13+ to prevent wraparound
- âœ… Monitor `age(datfrozenxid)` â€” alert at 1B, critical at 1.5B
- âœ… Set heartbeat on all connectors â€” slots lag even on idle tables
- âœ… Do NOT set `max.in.flight.requests=1` on Kafka >= 2.5 â€” unnecessary and kills throughput
- âœ… Deduplicate by `_offset_key`, not `ts_ms` â€” timestamps are not unique under high load

---

## ðŸ“ Repository Structure

```
cdc-enterprise-extreme/
â”‚
â”œâ”€â”€ README.md               â† You are here
â”œâ”€â”€ LICENSE                 â† MIT
â”œâ”€â”€ CONTRIBUTING.md         â† How to contribute
â”œâ”€â”€ CODE_OF_CONDUCT.md      â† Community standards
â”œâ”€â”€ SECURITY.md             â† Vulnerability reporting
â”‚
â”œâ”€â”€ docs/
â”‚   â”œâ”€â”€ architecture.md     â† Pipeline design and data flow
â”‚   â”œâ”€â”€ postgresql.md       â† PostgreSQL-specific guide
â”‚   â”œâ”€â”€ mysql.md            â† MySQL-specific guide
â”‚   â”œâ”€â”€ oracle.md           â† Oracle LogMiner guide
â”‚   â”œâ”€â”€ sqlserver.md        â† SQL Server guide
â”‚   â”œâ”€â”€ mongodb.md          â† MongoDB change streams guide
â”‚   â”œâ”€â”€ operations.md       â† Monitoring, DLQ, lifecycle
â”‚   â”œâ”€â”€ runbooks.md         â† Incident response procedures
â”‚   â””â”€â”€ performance.md      â† Tuning guide
â”‚
â”œâ”€â”€ diagrams/
â”‚   â””â”€â”€ architecture.mmd    â† Mermaid source diagrams
â”‚
â””â”€â”€ examples/
    â”œâ”€â”€ postgres-connector.json
    â”œâ”€â”€ oracle-connector.json
    â”œâ”€â”€ mysql-connector.json
    â”œâ”€â”€ sqlserver-connector.json
    â””â”€â”€ mongodb-connector.json
```

---

## ðŸ”¥ Post-Mortems Included

Real incidents, root causes, and lessons learned:

- **PM-001**: Inactive slot caused PostgreSQL wraparound â€” 47min write outage
- **PM-002**: Uncommunicated DDL broke 23 consumers simultaneously
- **PM-003**: Tombstone removed by `retention.ms` â€” ghost data survived in Silver layer
- **PM-004**: MongoDB resume token invalid after cluster migration
- **PM-005**: Oracle archive log deleted by RMAN before Debezium read â€” forced 8h re-snapshot
- **PM-006**: Oracle RAC redo log inaccessible after ASM mount failure

Full details in [docs/runbooks.md](docs/runbooks.md).

---

## ðŸ¤ Contributing

We welcome contributions! Please read [CONTRIBUTING.md](CONTRIBUTING.md) before opening a PR.

Areas where help is most needed:
- Additional database connectors (Cassandra, CockroachDB, Vitess)
- Kubernetes deployment examples (Strimzi, Confluent Operator)
- Additional post-mortems from the community
- Translations

---

## ðŸ“„ License

MIT â€” see [LICENSE](LICENSE).

---

## â­ Star History

If this project saved you from a production incident, please give it a star. It helps others find this resource.
