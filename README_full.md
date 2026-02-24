# Change Data Capture — Guia de Produção (2026)

> **Foco em produção real.** Este guia não é uma introdução conceitual — é um manual de implementação com garantias fortes, resiliência operacional e governança adequada.

---

## Sumário

1. [O Que CDC Realmente Significa](#1-o-que-cdc-realmente-significa)
2. [Versões Recomendadas](#2-versões-recomendadas)
3. [Quando Não Usar CDC Log-based](#3-quando-não-usar-cdc-log-based)
4. [Níveis de Maturidade do CDC](#4-níveis-de-maturidade-do-cdc)
5. [Arquitetura CDC com Kafka e Streaming](#5-arquitetura-cdc-com-kafka-e-streaming)
6. [Banco de Origem — Detalhamento Técnico](#6-banco-de-origem--detalhamento-técnico)
7. [Debezium — Configuração Correta](#7-debezium--configuração-correta)
8. [Formato de Evento no Tópico Kafka](#8-formato-de-evento-no-tópico-kafka)
9. [Camada Bronze — A Fundação Imutável](#9-camada-bronze--a-fundação-imutável)
10. [Camadas Silver e Gold](#10-camadas-silver-e-gold)
11. [Semânticas de Entrega e Idempotência](#11-semânticas-de-entrega-e-idempotência)
12. [Snapshot Inicial — Consistência e Estratégias](#12-snapshot-inicial--consistência-e-estratégias)
13. [Schema Evolution — DDL no Banco de Origem](#13-schema-evolution--ddl-no-banco-de-origem)
14. [Segurança, PII e Conformidade](#14-segurança-pii-e-conformidade)
15. [Modos de Deploy do Debezium](#15-modos-de-deploy-do-debezium)
16. [Testando Pipelines CDC](#16-testando-pipelines-cdc)
17. [Desafios Práticos e Como Mitigá-los](#17-desafios-práticos-e-como-mitigá-los)
18. [CDC em Arquiteturas de Data Lake](#18-cdc-em-arquiteturas-de-data-lake)
19. [CDC em Ambientes Cloud Gerenciados](#19-cdc-em-ambientes-cloud-gerenciados)
20. [Alta Disponibilidade e Multi-Região](#20-alta-disponibilidade-e-multi-região)
21. [Dimensionamento e Custos](#21-dimensionamento-e-custos)
22. [Monitoramento e Alertas](#22-monitoramento-e-alertas)
23. [Testes de Resiliência](#23-testes-de-resiliência)
24. [Matriz de Garantias](#24-matriz-de-garantias)
25. [Incidentes Reais Comuns](#25-incidentes-reais-comuns)
26. [Troubleshooting Comum](#26-troubleshooting-comum)
27. [Conclusão](#27-conclusão)

---

## 1. O Que CDC Realmente Significa

Change Data Capture (CDC) é a prática de capturar eventos de modificação de dados — INSERT, UPDATE e DELETE — diretamente dos mecanismos internos do banco de dados, preservando a ordem de commit, a consistência transacional e a capacidade de reprocessamento.

Ao contrário de abordagens baseadas em polling (consultas periódicas), o CDC orientado a log não depende de colunas de controle nem de timestamps. Ele lê o transaction log do banco — o mesmo registro que garante durabilidade e recuperação de falhas — transformando cada operação em um evento rastreável e reproduzível.

> ℹ **CDC maduro = log-based + checkpoint por offset + capacidade de replay desde qualquer ponto histórico.**

Em ambiente de produção, CDC precisa garantir:

- **Ordenação consistente:** eventos chegam na mesma sequência em que foram commitados no banco.
- **Offset rastreado:** a posição de leitura no log é persistida, garantindo retomada sem perda.
- **Replay confiável:** é possível reprocessar eventos históricos para reconstruir estados ou alimentar novos consumidores.
- **Idempotência:** processar o mesmo evento duas vezes deve produzir o mesmo resultado que processar uma vez.
- **Baixo impacto no banco:** a captura ocorre via leitura do log, sem sobrecarga nas transações originais.
- **Observabilidade operacional:** lag, saúde do slot e estado do conector são visíveis e alertáveis.

CDC mal implementado gera duplicidade silenciosa, perda de eventos, corrupção lógica, disk full no banco e snapshots forçados inesperados.

---

## 2. Versões Recomendadas

Evite stacks legadas. CDC é extremamente sensível ao comportamento interno de WAL/binlog, e versões antigas possuem bugs conhecidos de replication.

| Componente    | Versão recomendada | Observação |
|---------------|--------------------|------------|
| Kafka         | 3.7+               | `enable.idempotence=true` padrão desde 3.0 |
| Kafka Connect | 3.7+               | Worker distribuído; EOS v2 requer Connect 3.3+ |
| Debezium      | 2.5+               | Incremental snapshot com pause/resume via signal table |
| PostgreSQL    | 14+                | `pgoutput` nativo; `max_slot_wal_keep_size` obrigatório desde PG 13 |
| MySQL         | 8.0.34+ ou 8.4 LTS | `expire_logs_days` removido no 8.4 |
| Spark         | 3.5+               | `dropDuplicatesWithinWatermark` disponível desde 3.5 |
| Delta Lake    | 3.x                | Artefato: `delta-spark`. Liquid Clustering desde 3.1 |

> ⚠ **Migração Debezium 1.x → 2.x:** várias propriedades foram renomeadas. `database.dbname` → `database.names` e `database.history.*` → `schema.history.internal.*`. Valide todas as configurações antes da migração.

---

## 3. Quando Não Usar CDC Log-based

**Evite CDC log-based quando:**

- **A tabela não tem chave primária definida.** O Debezium depende da PK para compor a message key do Kafka. Tabelas sem PK podem ser capturadas com `REPLICA IDENTITY FULL` no PostgreSQL, mas isso aumenta drasticamente o volume do WAL.
- **A fonte de dados não expõe o transaction log.** Serviços SaaS (Salesforce, HubSpot, Stripe) não oferecem acesso ao log. Use conectores baseados em API (Airbyte, Fivetran) ou webhooks nativos.
- **O volume é baixo e a latência tolerada é alta.** Para tabelas pequenas com poucas escritas e SLA de horas, um simples job de polling com `updated_at` atende com menos complexidade.
- **O banco não suporta log-based CDC.** SQLite, Access e bancos embarcados não expõem transaction log.
- **O schema muda com altíssima frequência de forma destrutiva.** Pipelines com `DROP COLUMN` e `RENAME COLUMN` recorrentes exigem coordenação contínua.

> ℹ **O critério decisivo:** se você precisa de baixa latência, captura de DELETEs, rastreabilidade completa e o banco suporta — use CDC log-based. Nos demais casos, avalie a alternativa mais simples.

---

## 4. Níveis de Maturidade do CDC

| Nível | Estratégia | Captura DELETE? | Ordem Transacional? | Impacto no Banco |
|---|---|---|---|---|
| 1 — Watermark | Coluna `updated_at` | ❌ Não | ❌ Não | Baixo |
| 2 — Change Tracking¹ | Metadados SQL Server | ✅ Sim | Parcial | Baixo |
| 3 — Trigger-based | Triggers de auditoria | ✅ Sim | Parcial | Alto |
| 4 — Log-based | WAL / Binlog / CDC nativo | ✅ Sim | ✅ Sim | Mínimo |

> ¹ Change Tracking é exclusivo do SQL Server. Registra *quais* linhas mudaram, mas não captura os valores anteriores (`before`). Não confundir com o CDC nativo (nível 4).

### 4.1 Change Tracking vs CDC Nativo (SQL Server)

- **Change Tracking (Nível 2):** registra apenas *quais* linhas foram alteradas sem o estado `before`.
- **CDC nativo (Nível 4):** lê o transaction log com estado completo `before` e `after`.

> ℹ **O Debezium para SQL Server utiliza o CDC nativo (`sys.sp_cdc_enable_table`), e não o Change Tracking.**

---

## 5. Arquitetura CDC com Kafka e Streaming

```
[ Banco de Dados: PostgreSQL / MySQL / SQL Server / MongoDB / Oracle ]
          |
          v
[ Debezium Connector ]  ←  lê o transaction log continuamente
          |
          v
[ Kafka Topic por Tabela ]  ex: dbserver1.public.orders
          |
    +------+-------+-------+
    |              |       |
    v              v       v
[ Bronze Lake ] [Flink]  [ML Pipeline]
    |
    v
[ Silver: merge/dedup idempotente ]
    |
    v
[ Gold: modelo analítico ]
```

### 5.1 Kafka como Backbone — Configurações-Chave

- **Particionamento por chave primária:** o Debezium usa a PK da tabela como *message key*, garantindo que todas as mudanças de uma mesma linha caiam na mesma partição. Considerações:
  - **Hash estável:** nunca altere o tipo da PK sem recalcular o mapeamento — eventos podem ir para partições diferentes.
  - **Hot partitions:** PKs com baixa cardinalidade podem gerar partições com volume desproporcional. Monitore throughput por partição.
  - **Chave composta:** quando a PK é composta (ex: `(tenant_id, order_id)`), o consumer deve usar a chave completa para ordenação.
  - **Mudança de PK em produção:** é equivalente a reprocessamento full. Planeje como uma migração completa.
- **Schema Registry:** integração com Confluent Schema Registry ou AWS Glue para versionamento de schemas.
- **Idempotência do produtor:** a partir do Kafka 3.0, `enable.idempotence=true` é o padrão.

> ℹ **Alterar a message key sem cuidado quebra a garantia de ordering por entidade.**

---

## 6. Banco de Origem — Detalhamento Técnico

### 6.1 PostgreSQL — Logical Decoding + WAL

#### Requisitos

```sql
ALTER SYSTEM SET wal_level                       = logical;
ALTER SYSTEM SET max_replication_slots            = 10;
ALTER SYSTEM SET max_wal_senders                  = 10;
ALTER SYSTEM SET max_logical_replication_workers  = 10;
ALTER SYSTEM SET max_worker_processes             = 20;
ALTER SYSTEM SET max_slot_wal_keep_size           = '20GB';
```

#### Publicação e Replication Slot

```sql
-- Opção A: publicação por tabela
CREATE PUBLICATION dbz_publication FOR TABLE public.orders;

-- Opção B: todas as tabelas (novas entram automaticamente)
CREATE PUBLICATION dbz_publication FOR ALL TABLES;

-- Criar replication slot
SELECT pg_create_logical_replication_slot('dbz_slot', 'pgoutput');

-- Monitorar lag
SELECT slot_name, active,
       pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes
FROM   pg_replication_slots;
```

> ✗ **Risco crítico — Disk Full:** se o conector parar, o `restart_lsn` não avança e o WAL acumula indefinidamente. Configure alertas em `lag_bytes`. Se o conector ficar parado por longo período, drope o slot e recrie via snapshot incremental.

> ⚠ **PostgreSQL 13+:** configure `max_slot_wal_keep_size` como safety net.

---

### 6.2 MySQL / MariaDB — Binlog ROW Format

```ini
# MySQL 5.7
[mysqld]
server-id        = 1
log_bin          = ON
binlog_format    = ROW
binlog_row_image = FULL
expire_logs_days = 7           # deprecated desde 8.0

# MySQL 8.0+
binlog_expire_logs_seconds = 604800   # 7 dias
```

> ⚠ **MySQL 8.4 LTS:** `expire_logs_days` foi removido. Use exclusivamente `binlog_expire_logs_seconds`.

#### Cálculo da retenção de binlog

```
retenção mínima (s) = tempo_máximo_inatividade + janela_SLA_recuperação + margem_50%
```

#### GTID — Cuidados

- **Nunca manipule `gtid_purged` manualmente** após o conector estar configurado.
- **Nunca force purge de binlogs** sem validar que o offset foi consumido.
- Em failover para réplica, o GTID garante consistência — mas valide que a réplica está em dia.

---

### 6.3 SQL Server — CDC Nativo

```sql
EXEC sys.sp_cdc_enable_db;

EXEC sys.sp_cdc_enable_table
    @source_schema     = 'dbo',
    @source_name       = 'orders',
    @role_name         = NULL,
    @supports_net_changes = 0;  -- Debezium usa "all changes"; net changes não necessário
```

> ⚠ **Retenção CDC:** o padrão é 3 dias. Ajuste conforme o SLA:
> ```sql
> EXEC sys.sp_cdc_change_job @job_type = 'cleanup', @retention = 10080;
> ```

---

### 6.4 MongoDB — Change Streams

```javascript
const changeStream = db.orders.watch([
    { $match: { operationType: { $in: ['insert', 'update', 'delete'] } } }
], {
    fullDocument: 'updateLookup',
    resumeAfter: savedResumeToken
});
```

> ℹ **MongoDB exige replica set** para Change Streams, mesmo em instâncias únicas.

> ⚠ **Resume Token é opaco e deve ser persistido integralmente.** Nunca tente parsear ou reconstruir. O campo `clusterTime` **não garante unicidade** — o Resume Token é a única referência segura de checkpoint.

---

## 7. Debezium — Configuração Correta

### 7.1 PostgreSQL

```json
{
  "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
  "database.hostname": "postgres",
  "database.port": "5432",
  "database.user": "debezium",
  "database.password": "${file:/opt/connect/secrets.properties:pg.password}",
  "database.dbname": "appdb",
  "plugin.name": "pgoutput",
  "publication.name": "dbz_publication",
  "slot.name": "dbz_slot",
  "snapshot.mode": "initial",
  "tombstones.on.delete": "true",
  "topic.prefix": "dbserver1",
  "signal.data.collection": "public.debezium_signals",
  "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
  "schema.history.internal.kafka.topic": "schema-changes.orders"
}
```

### 7.2 MySQL / SQL Server (Debezium 2.x)

> ⚠ **Debezium 2.x:** `database.dbname` foi substituído por `database.names`. Usar `database.dbname` pode causar falha silenciosa.

```json
{
  "connector.class": "io.debezium.connector.sqlserver.SqlServerConnector",
  "database.hostname": "sqlserver",
  "database.port": "1433",
  "database.user": "debezium",
  "database.password": "${file:/opt/connect/secrets.properties:ss.password}",
  "database.names": "appdb",
  "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
  "schema.history.internal.kafka.topic": "schema-changes.orders",
  "table.include.list": "dbo.orders",
  "topic.prefix": "dbserver1",
  "tombstones.on.delete": "true",
  "signal.data.collection": "dbo.debezium_signals"
}
```

> ℹ **`tombstones.on.delete=true`** publica um evento tombstone (valor `null`) após cada DELETE, necessário para tópicos com `cleanup.policy=compact`.

---

## 8. Formato de Evento no Tópico Kafka

Payload completo de um UPDATE:

```json
{
  "payload": {
    "before": { "id": 42, "status": "open", "total": 150.00 },
    "after":  { "id": 42, "status": "paid", "total": 150.00 },
    "source": {
      "connector": "postgresql",
      "db": "ecommerce",
      "table": "orders",
      "lsn": 12345678,
      "ts_ms": 1710000000000,
      "txId": 789
    },
    "op": "u",
    "ts_ms": 1710000001000
  }
}
```

> ⚠ **Os campos dentro de `source` variam por conector:**
>
> | Banco | Campo de posição | Campo de transação |
> |---|---|---|
> | PostgreSQL | `lsn` (inteiro) | `txId` |
> | MySQL / MariaDB | `file` + `pos` (composto) | `gtid` (se ativo) |
> | SQL Server | `change_lsn` + `commit_lsn` | — |
> | MongoDB | `resume_token` (string BSON) | `clusterTime` (não único) |
> | Oracle | `commit_scn` + `scn` + `xid` | `xid` |

**Valores de `op`:** `c` = insert · `u` = update · `d` = delete · `r` = snapshot/read

**Regra de ouro:** `after` é `null` para DELETE e `before` é `null` para INSERT. Qualquer código que acesse `b.after.id` sem verificar o tipo de operação falhará silenciosamente em DELETEs.

> ✗ **Nunca acesse `b.after.id` diretamente em merges** — use `CASE WHEN b.op = 'd' THEN b.before.id ELSE b.after.id END`.

---

## 9. Camada Bronze — A Fundação Imutável

A Bronze armazena eventos exatamente como chegaram, sem transformação, deduplicação ou merge. Cada registro deve conter:

- **Operação (`op`):** `c`, `u`, `d` ou `r`.
- **Estado `before`:** imagem antes da operação (`null` para INSERTs).
- **Estado `after`:** imagem após a operação (`null` para DELETEs).
- **Offset:** posição exata no transaction log.
- **Timestamp de origem (`source.ts_ms`).**
- **Metadados do conector:** versão, nome do servidor, transaction ID, tabela.

> ℹ **Nunca sobrescreva dados na Bronze.** A imutabilidade garante o replay total.

---

## 10. Camadas Silver e Gold

### 10.1 Silver — Merge Incremental Correto e Idempotente

**✗ Bug comum — acesso direto a `after.id`:**

```python
# ❌ INCORRETO — falha silenciosamente em DELETEs (after = null)
.merge(batch_df.alias('b'), condition='s.id = b.after.id')
```

**Solução correta — CASE WHEN por tipo de operação, com deduplicação robusta por banco:**

```python
from delta.tables import DeltaTable
from pyspark.sql import functions as F


def build_offset_key(df, source_type: str):
    """
    Constrói chave de deduplicação conforme o banco de origem.
    PostgreSQL  : source.lsn  (inteiro monotônico)
    SQL Server  : source.change_lsn  (NÃO source.lsn — retorna null)
    MySQL       : source.file + ':' + source.pos  (pos reinicia por arquivo)
    MongoDB     : source.resume_token  (clusterTime não garante unicidade)
    Oracle      : source.commit_scn + ':' + source.scn + ':' + source.xid
    """
    if source_type == 'postgresql':
        return df.withColumn('_offset_key', F.col('source.lsn').cast('string'))
    elif source_type == 'sqlserver':
        return df.withColumn('_offset_key', F.col('source.change_lsn').cast('string'))
    elif source_type in ('mysql', 'mariadb'):
        return df.withColumn(
            '_offset_key',
            F.concat_ws(':', F.col('source.file'), F.col('source.pos').cast('string'))
        )
    elif source_type == 'mongodb':
        return df.withColumn('_offset_key', F.col('source.resume_token'))
    elif source_type == 'oracle':
        return df.withColumn(
            '_offset_key',
            F.concat_ws(':',
                F.col('source.commit_scn').cast('string'),
                F.col('source.scn').cast('string'),
                F.col('source.xid')
            )
        )
    else:
        raise ValueError(f"source_type não suportado: {source_type}")


def upsert_to_silver(batch_df, batch_id, source_type='postgresql'):
    silver = DeltaTable.forPath(spark, '/silver/orders')

    # 1. Adicionar chave de offset conforme o banco
    deduped = build_offset_key(batch_df, source_type)

    # 2. Deduplicação dentro do batch
    #    ATENÇÃO — dropDuplicates mantém o PRIMEIRO registro.
    #    Em Structured Streaming contínuo, use dropDuplicatesWithinWatermark
    #    (Spark 3.5+) para evitar estado ilimitado:
    #
    #      deduped = (
    #          deduped
    #          .withWatermark('event_time', '2 hours')
    #          .dropDuplicatesWithinWatermark(['_offset_key'])
    #      )
    deduped = deduped.dropDuplicates(['_offset_key'])

    # 3. MERGE com CASE WHEN para DELETE safety
    silver.alias('s').merge(
        deduped.alias('b'),
        condition='''
            s.id = CASE
                WHEN b.op = 'd' THEN b.before.id
                ELSE b.after.id
            END
        '''
    ).whenMatchedDelete(
        condition="b.op = 'd'"                             # DELETE primeiro
    ).whenMatchedUpdate(
        condition="b.op IN ('u', 'c')",                    # UPDATE + replay idempotente
        set={
            'status':      'b.after.status',
            'total':       'b.after.total',
            'updated_at':  'b.source.ts_ms',
            '_offset_key': 'b._offset_key'
        }
    ).whenNotMatchedInsert(
        condition="b.op != 'd'",
        values={
            'id':          'b.after.id',
            'status':      'b.after.status',
            'total':       'b.after.total',
            'updated_at':  'b.source.ts_ms',
            '_offset_key': 'b._offset_key'
        }
    ).execute()


bronze_stream = spark.readStream.format('delta').load('/bronze/orders')
bronze_stream.writeStream \
    .foreachBatch(lambda df, bid: upsert_to_silver(df, bid, source_type='postgresql')) \
    .option('checkpointLocation', '/checkpoints/silver_orders') \
    .start()
```

> ⚠ **No MySQL, a chave de offset é composta (`file:pos`)** — usar somente `pos` gera falsos positivos no rollover de arquivo.

> ⚠ **No MongoDB, use `source.resume_token`**, não `source.ord`. O `ord` não garante unicidade entre sessões.

> ⚠ **No SQL Server, use `source.change_lsn`**, não `source.lsn`. O campo genérico `source.lsn` **não existe** no payload do SQL Server e retorna `null` silenciosamente.

### 10.2 Gold — Modelo Analítico

A Gold entrega agregações, dimensões e fatos modelados para BI, APIs ou ML. Pode ser reconstruída a qualquer momento a partir da Bronze.

---

## 11. Semânticas de Entrega e Idempotência

### 11.1 At-Least-Once — Padrão Debezium

O Debezium garante *at-least-once delivery*: em caso de falha, alguns eventos podem ser republicados. Idempotência é implementada via:

- **Deduplicação por offset:** use `_offset_key` adequada ao banco (seção 10.1).
- **MERGE com PK:** MERGE é naturalmente idempotente.
- **Tombstone para DELETEs:** o Debezium publica dois eventos para DELETE — o evento com `op='d'` e um tombstone (`null`). A tombstone é retida por `delete.retention.ms` (padrão: 24h).

### 11.2 Exactly-Once com Kafka Transactions

Exactly-once requer **todas** as condições simultaneamente:

- Produtor transacional (`enable.idempotence=true`, padrão desde Kafka 3.0)
- Consumidor com `isolation.level=read_committed`
- Kafka Connect em modo exactly-once (Connect 3.3+)
- Sink idempotente

```properties
# kafka-connect worker.properties
exactly.once.source.support=enabled
```

> ℹ **Apache Flink com Delta Lake sink** oferece exactly-once nativo via two-phase commit.

> ℹ **Kafka Streams:** use `processing.guarantee=exactly_once_v2` (desde Kafka 2.6).

---

## 12. Snapshot Inicial — Consistência e Estratégias

### 12.1 O Problema de Consistência

> ⚠ **`snapshot.mode=schema_only` — perda de dados históricos:** este modo ignora todos os dados existentes. Para Silver completa, faça backfill separado antes de ativar em modo `schema_only`.

### 12.2 Snapshot Tradicional (com Lock)

O Debezium adquire lock de leitura durante o snapshot. Após o snapshot, retoma do LSN registrado. Em tabelas de alta concorrência, prefira snapshot incremental.

### 12.3 Snapshot Incremental (Recomendado)

```json
{
  "snapshot.mode": "initial",
  "incremental.snapshot.chunk.size": 1024,
  "signal.data.collection": "debezium.signals"
}
```

```sql
-- Disparar snapshot incremental via tabela de sinais
INSERT INTO debezium.signals (id, type, data)
VALUES ('snap-001', 'execute-snapshot',
        '{"data-collections": ["public.orders"]}');
```

> ℹ **Seguro para produção:** não bloqueia a tabela, pode ser pausado e retomado.

### 12.4 Snapshot via Exportação Paralela

Para tabelas muito grandes (centenas de GBs), exporte via `COPY TO` / `mysqldump`, importe na Bronze registrando o LSN, e ative o Debezium a partir desse ponto.

---

## 13. Schema Evolution — DDL no Banco de Origem

| Operação DDL | PostgreSQL | MySQL | SQL Server |
|---|---|---|---|
| `ADD COLUMN` | ✅ Detectado | ✅ Detectado | ✅ Detectado |
| `DROP COLUMN` | ✅ Detectado | ✅ Detectado | ✅ Detectado |
| `RENAME COLUMN` | ⚠ Pode exigir reset | ✅ Detectado | ⚠ Exige atenção |
| `CHANGE TYPE` | ⚠ Pode quebrar consumers | ⚠ Pode quebrar consumers | ⚠ Pode quebrar consumers |

### 13.1 Formato de Serialização

| Formato | Schema enforcement | Evolução compatível | Overhead | Indicado para |
|---|---|---|---|---|
| **Avro** | Forte (via Registry) | BACKWARD/FORWARD/FULL | Baixo | Produção |
| **Protobuf** | Forte (via Registry) | Field numbers | Baixo | Produção (gRPC) |
| **JSON Schema** | Moderado | Possível | Alto | Dev / integração JSON |
| **JSON sem Registry** | Nenhum | Sem garantia | Alto | ❌ Não recomendado |

> ⚠ **Avro e nullable:** campos só podem ser `null` se declarados como union `["null", "tipo"]` com default `null`.

### 13.2 Política de Compatibilidade

| Operação | Compatível com BACKWARD? | Como fazer |
|---|---|---|
| `ADD COLUMN NULL` ou com `DEFAULT` | ✅ Sim | Padrão seguro |
| `ADD COLUMN NOT NULL` sem default | ❌ Não | Adicionar com `DEFAULT` primeiro |
| `DROP COLUMN` obrigatório | ❌ Não | Deprecar → migrar → remover |
| `RENAME COLUMN` | ❌ Não | Adicionar nova, migrar, remover antiga |
| `CHANGE TYPE` compatível | ⚠ Depende | Validar no Registry |
| `CHANGE TYPE` incompatível | ❌ Não | Novo tópico + migração |

```bash
# Testar compatibilidade antes de registrar
POST /compatibility/subjects/dbserver1.public.orders-value/versions/latest
{"schema": "...novo schema Avro..."}
# Response: {"is_compatible": true}
```

> ⚠ **`DROP COLUMN` e `RENAME COLUMN` são operações destrutivas** — nenhuma política permite sem coordenação explícita.

---

## 14. Segurança, PII e Conformidade

### 14.1 Mascaramento de PII no Conector

```json
{
  "column.mask.hash.SHA-256.with.salt.mySalt": "dbo.customers.email,dbo.customers.cpf",
  "column.mask.with.12.chars": "dbo.payments.card_number,dbo.payments.cvv",
  "column.exclude.list": "dbo.customers.raw_password"
}
```

> ✗ **Nunca confie apenas no controle de acesso ao tópico para proteção de PII.** O mascaramento deve ocorrer no conector — antes da publicação.

### 14.2 Segurança em Trânsito e Repouso

- **TLS banco→conector:** `sslmode=verify-full` (PG) ou `useSSL=true&requireSSL=true` (MySQL).
- **TLS conector→Kafka:** `security.protocol=SSL` + keystores no worker.
- **Criptografia em repouso:** SSE-S3, SSE-KMS/CMK, ADLS encryption ou Parquet column encryption.
- **Rotação de credenciais:** use secret manager (AWS Secrets Manager, Vault, Azure Key Vault).

### 14.3 Controle de Acesso

```bash
# Apenas o conector pode produzir
kafka-acls --add --allow-principal User:debezium-connector \
  --operation Write --topic dbserver1.public.orders

# Apenas consumidores autorizados podem ler
kafka-acls --add --allow-principal User:silver-pipeline \
  --operation Read --topic dbserver1.public.orders --group silver-consumer-group
```

### 14.4 Direito ao Esquecimento (LGPD / GDPR)

- **Crypto shredding:** criptografe PII com chave por usuário **antes de gravar na Bronze**. Ao solicitar exclusão, destrua a chave.
- **Tombstone + reprocessamento:** publique evento com campos PII nulos e reconstrua a Silver.
- **Segregação de PII:** armazene PII em tabela separada referenciada por token.

## 15. Modos de Deploy do Debezium

### 15.1 Kafka Connect (Padrão)

O modo mais utilizado em produção. O Debezium roda como conector dentro de um cluster Kafka Connect.

**Indicado para:** times com Kafka Connect já operacional, múltiplos conectores, escalabilidade horizontal.

### 15.2 Debezium Server (Standalone)

Aplicação standalone que publica eventos diretamente em destinos sem Kafka:

```properties
debezium.source.connector.class=io.debezium.connector.postgresql.PostgresConnector
debezium.source.database.hostname=postgres
debezium.source.database.port=5432
debezium.source.database.user=debezium
debezium.source.database.password=${DB_PASSWORD}
debezium.source.database.dbname=ecommerce
debezium.source.table.include.list=public.orders

debezium.sink.type=kinesis
debezium.sink.kinesis.region=us-east-1
debezium.sink.kinesis.stream.name=cdc-orders
```

| Destino | Tipo | Caso de Uso |
|---|---|---|
| Kafka | Streaming | Pipeline completo |
| Amazon Kinesis | Streaming gerenciado | AWS sem Kafka |
| Google Pub/Sub | Streaming gerenciado | GCP |
| Azure Event Hubs | Streaming gerenciado | Azure |
| Redis Streams | In-memory | Latência ultra-baixa |
| HTTP/Webhook | Genérico | Integrações customizadas |

### 15.3 Debezium Embedded Library

Para CDC dentro de uma aplicação JVM:

```java
DebeziumEngine<ChangeEvent<String, String>> engine = DebeziumEngine
    .create(Json.class)
    .using(props)
    .notifying(record -> processChangeEvent(record))
    .build();

ExecutorService executor = Executors.newSingleThreadExecutor();
executor.execute(engine);
```

**Indicado para:** microserviços que precisam reagir a mudanças do próprio banco (cache invalidation, event sourcing local).

---

## 16. Testando Pipelines CDC

### 16.1 Testes de Unidade — Validação do Schema

```python
import pytest
from pyspark.sql import SparkSession

@pytest.fixture(scope='session')
def spark():
    return SparkSession.builder \
        .config('spark.jars.packages', 'io.delta:delta-spark_2.12:3.2.0') \
        .config('spark.sql.extensions', 'io.delta.sql.DeltaSparkSessionExtension') \
        .config('spark.sql.catalog.spark_catalog', 'org.apache.spark.sql.delta.catalog.DeltaCatalog') \
        .getOrCreate()

DELETE_EVENT = {
    'op': 'd',
    'before': {'id': 42, 'status': 'paid', 'total': 150.0},
    'after': None,
    'source': {'lsn': 99999, 'ts_ms': 1710000000000}
}

def test_delete_event_id_extraction(spark):
    df = spark.createDataFrame([DELETE_EVENT])
    result = df.selectExpr(
        "CASE WHEN op = 'd' THEN before.id ELSE after.id END AS entity_id"
    )
    assert result.first()['entity_id'] == 42
```

### 16.2 Testes de Integração com Testcontainers

```python
from testcontainers.postgres import PostgresContainer
from testcontainers.kafka import KafkaContainer

@pytest.fixture(scope='module')
def postgres():
    with PostgresContainer('postgres:15') \
            .with_command('postgres -c wal_level=logical') as pg:
        yield pg

def test_insert_propagates_to_kafka(postgres, kafka_consumer):
    conn = psycopg2.connect(postgres.get_connection_url())
    cur = conn.cursor()
    cur.execute("INSERT INTO orders (id, status) VALUES (1, 'open')")
    conn.commit()

    message = next(kafka_consumer)
    payload = message.value['payload']

    assert payload['op'] == 'c'
    assert payload['after']['id'] == 1
    assert payload['before'] is None
```

### 16.3 Cenários Críticos de Regressão

| Cenário | O que testar | Por que é crítico |
|---|---|---|
| DELETE event | `after` é `null`; merge usa `before.id` | Bug mais comum |
| Replay de eventos | Mesmo offset → mesmo estado | Idempotência |
| Schema change | Novo campo aparece; consumers não quebram | Schema evolution |
| Conector reiniciado | Nenhum evento perdido | At-least-once |
| Deduplicação MySQL | `file+pos` no rollover de arquivo | Idempotência multi-banco |
| Deduplicação MongoDB | `resume_token` como chave única | Idempotência MongoDB |

---

## 17. Desafios Práticos e Como Mitigá-los

### 17.1 Replication Slot Bloat — PostgreSQL

- Monitore `pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)`.
- Configure `max_slot_wal_keep_size` (PG 13+).
- Slot inativo prolongado: drope imediatamente e recrie via snapshot incremental.

### 17.2 Schema Evolution

Schema Registry com política `BACKWARD` ou `FULL`. Remoção e renomeação exigem coordenação — consulte a seção 13.

### 17.3 Consumer Lag

Métricas-chave: `kafka_consumer_lag`, `debezium_metrics_MilliSecondsBehindSource`, `lag_bytes` nos replication slots.

### 17.4 Tombstones e Log Compaction

Em tópicos com `cleanup.policy=compact`, tombstones são retidas por `delete.retention.ms` (padrão: 24h). Consumidores devem tratar tombstones explicitamente — ignorá-las pode deixar dados desatualizados.

### 17.5 Tabelas sem Chave Primária

```sql
-- PostgreSQL: captura completa em tabela sem PK (use com cautela)
ALTER TABLE legacy_table REPLICA IDENTITY FULL;
```

> ⚠ **Tabelas sem PK são um sinal de problema de modelagem.** Adicione uma PK antes de configurar o CDC.

---

## 18. CDC em Arquiteturas de Data Lake

### 18.1 Comparativo dos Formatos

- **Delta Lake:** `MERGE INTO` nativo, schema evolution, time travel. Artefato: `io.delta:delta-spark_2.12:3.x.x`.
- **Apache Iceberg:** multi-engine (Spark, Flink, Trino). MERGE via merge-on-read ou copy-on-write.
- **Apache Hudi:** orientado a CDC com MOR e COW. Incremental queries built-in.

### 18.2 Delta Lake — Controle de Concorrência

```python
spark.conf.set("spark.databricks.delta.retryWriteConflicts.enabled", "true")
spark.conf.set("spark.databricks.delta.maxCommitAttempts", "10")
```

#### VACUUM

```sql
VACUUM silver.orders RETAIN 168 HOURS;
```

> ⚠ **Não reduza abaixo de 7 dias** — pode quebrar time travel e leituras concorrentes.

#### OPTIMIZE e ZORDER

```sql
OPTIMIZE silver.orders ZORDER BY (customer_id, status);

-- Delta Lake 3.1+: Liquid Clustering
ALTER TABLE silver.orders CLUSTER BY (customer_id, status);
```

> ℹ **Automatize `OPTIMIZE` em produção.** Em Databricks, use `optimizeWrite=true` e `autoCompact=true`.

### 18.3 Hudi — MOR vs COW

- **COW:** cada atualização reescreve o arquivo. Leituras rápidas, escritas custosas.
- **MOR:** atualizações em logs delta, mesclados na leitura. Ingestão de baixa latência.

> ℹ **Hudi MOR em produção:** use compactação **assíncrona** para evitar degradação de leitura.

### 18.4 Time Travel

```sql
-- Delta Lake
SELECT * FROM silver.orders VERSION AS OF 10;

-- Iceberg
SELECT * FROM silver.orders FOR SYSTEM_TIME AS OF TIMESTAMP '2024-03-15 10:00:00';

-- Hudi
SELECT * FROM silver.orders WHERE _hoodie_commit_time <= '20240315100000';
```

---

## 19. CDC em Ambientes Cloud Gerenciados

| Serviço | Provider | Bancos | Destinos | Observação |
|---|---|---|---|---|
| AWS DMS | Amazon | Oracle, SS, MySQL, PG, Mongo | S3, Redshift, Kinesis | Latência pode subir com Full LOB mode |
| Confluent Cloud | Confluent | PG, MySQL, SS, Mongo, Oracle | Kafka nativo | Debezium gerenciado; latência < 1s |
| Azure Data Factory | Microsoft | SS/Azure SQL, PG, MySQL, Oracle | ADLS, Synapse | Change Tracking para Azure SQL |
| Google Datastream | Google | Oracle, MySQL, PG | BigQuery, GCS, Spanner | Serverless; latência < 10s |
| Fivetran / Airbyte | SaaS / OSS | 50+ conectores | Warehouses, lakes | Airbyte OSS; Fivetran gerenciado |

---

## 20. Alta Disponibilidade e Multi-Região

- **MirrorMaker 2:** replicação cross-region com sincronização de offsets.
- **DR testado periodicamente:** capacidade de recuperar de DR não testado é zero.
- **Nunca dependa de cluster único** em produção crítica.

```properties
clusters = source, target
source.bootstrap.servers = source-kafka:9092
target.bootstrap.servers = target-kafka:9092
source->target.enabled = true
source->target.topics = dbserver1\.*
source->target.sync.group.offsets.enabled = true
```

---

## 21. Dimensionamento e Custos

### 21.1 Estimativa de Throughput

- **Partições:** idealmente igual ao número de consumidores paralelos.
- **Retenção:** defina com base na janela de reprocessamento.
- **Tamanho do evento:** 200 bytes a vários KB.
- **Cálculo:** 10.000 eventos/s × 1 KB = 10 MB/s ≈ 86 GB/dia.

### 21.2 Exemplo

Tabela com 1M atualizações/dia × 1 KB:
- ~11,6 eventos/s ≈ 11,6 KB/s
- 7 dias de retenção: ~7 GB no Kafka
- 10 tabelas similares: ~70 GB
- Custo estimado (AWS MSK, 3 brokers m5.large): ~$600/mês + EBS

---

## 22. Monitoramento e Alertas

### 22.1 Métricas Debezium (JMX)

- `debezium_metrics_MilliSecondsBehindSource` — lag
- `debezium_metrics_TotalNumberOfEventsSeen` — total de eventos
- `debezium_metrics_QueueRemainingCapacity` — capacidade restante

### 22.2 Métricas Kafka

- Consumer lag por grupo: `kafka_consumer_lag`
- Taxa: `MessagesInPerSec`, `BytesInPerSec`
- Under-replicated partitions — saúde do cluster

### 22.3 Métricas do Banco

- **PostgreSQL:** `lag_bytes` em `pg_replication_slots` (alerte se > 50 GB ou > 30 min)
- **MySQL:** espaço dos binlogs e arquivo mais antigo necessário
- **SQL Server:** tamanho da tabela de captura CDC

### 22.4 Métricas Spark

- **Batch duration:** se cresce consistentemente, o pipeline está perdendo
- **State store size:** crescimento contínuo indica ausência de watermark
- **Checkpoint health:** checkpoints lentos atrasam o próximo micro-batch

> ⚠ **State store sem watermark = OOM inevitável.** Use `withWatermark` + `dropDuplicatesWithinWatermark` (Spark 3.5+):
> ```python
> deduped = (
>     parsed_df
>     .withWatermark('event_time', '2 hours')
>     .dropDuplicatesWithinWatermark(['_offset_key'])
> )
> ```

### 22.5 Alertas Recomendados

| Alerta | Condição | Ação |
|---|---|---|
| Replication slot lag | > 50 GB ou parado > 30 min | Investigar conector; dropar slot |
| Consumer lag | > 10 min (ou SLA) | Escalar consumidores |
| Conector parado | Status != RUNNING | Reiniciar via API |
| Queda de throughput | Redução > 50% | Verificar banco e conectores |
| Binlog/WAL perto da expiração | > 80% da retenção | Aumentar retenção |

---

## 23. Testes de Resiliência

Simule com Testcontainers ou staging:

1. **Banco indisponível:** parar e reiniciar. Verificar retomada do offset.
2. **Kafka fora do ar:** reconexão com backoff exponencial.
3. **Reinicialização do conector:** offsets restaurados sem perda.
4. **Corrupção de offset:** recuperação via snapshot.
5. **Sobrecarga:** picos de inserções; lag controlado.
6. **Deduplicação:** mensagens duplicadas não aplicadas na Silver.

**Ferramentas:** Testcontainers, k6/JMeter, Chaos Mesh/Gremlin.

> ℹ **Automatize em CI.** Resiliência é validada continuamente, não uma vez.

---

## 24. Matriz de Garantias

| Camada | Garantia | Responsabilidade |
|---|---|---|
| Banco | Ordem transacional | WAL / Binlog / CDC nativo |
| Debezium | Captura at-least-once | Offset + replication slot |
| Kafka | At-least-once (EOS se transacional) | Broker + retenção |
| Spark | Processamento incremental | Checkpoint + foreachBatch |
| Delta/Iceberg/Hudi | Idempotência via MERGE | Schema + PK + _offset_key |

---

## 25. Incidentes Reais Comuns

| Problema | Causa Raiz | Mitigação |
|---|---|---|
| Disk full no banco | Replication slot não consumido / Binlog acumulado | Alertas + `max_slot_wal_keep_size` |
| Snapshot inesperado | Binlog/WAL expirado com conector parado | Aumentar retenção; monitorar |
| Duplicidade silenciosa | Rebalance + sink não idempotente | MERGE + dedup por offset |
| Quebra de consumers | DDL destrutivo sem coordenação | Schema Registry + ciclo explícito |
| Ordering quebrado | PK alterada ou message key incorreta | Nunca alterar PK sem plano |
| DELETE não aplicado | Merge usando `after.id` sem CASE WHEN | Usar `CASE WHEN b.op = 'd'` |
| OOM no Spark | `dropDuplicates` sem watermark | `dropDuplicatesWithinWatermark` |
| Conector não inicia pós-2.x | `database.dbname` → `database.names` | Validar todas as props |
| Silver corrompida pós-PK | PK antiga e nova = entidades distintas | Alterar PK = reprocessamento full |

### 25.1 Riscos Específicos Enterprise

#### Clock Skew entre Regiões

Em pipelines multi-região, **nunca use `source.ts_ms` como critério único de ordenação**. Use LSN/offset por banco como critério primário.

#### Reprocessamento após Retenção Kafka Expirar

```python
spark.readStream.format("kafka") \
    .option("subscribe", "orders") \
    .option("startingOffsets", "earliest") \
    .option("failOnDataLoss", "false")
```

> ⚠ `failOnDataLoss=false` permite continuar mesmo com gap, mas eventos perdidos não são reprocessados. A Bronze imutável é o ponto de replay.

#### Large Transactions

Transações que modificam milhões de linhas geram batches enormes. Mitigue com `max.poll.records`, monitore `pg_stat_activity` e quebre operações bulk em batches menores.

---

## 26. Troubleshooting Comum

### 26.1 Conector não inicia

- Verifique host, porta e credenciais.
- Confirme `wal_level=logical` (PG) ou `binlog_format=ROW` (MySQL).
- No Debezium 2.x: use `database.names`, não `database.dbname`.
- No Kafka Connect: verifique acesso aos tópicos de offset e histórico.

### 26.2 Eventos não chegam ao Kafka

- Verifique `GET /connectors/{name}/status`.
- Monitore lag no banco.
- Confirme tópico de destino criado com partições suficientes.

### 26.3 DELETEs não aplicados na Silver

- O merge provavelmente usa `after.id` diretamente. Corrija com `CASE WHEN b.op = 'd' THEN b.before.id ELSE b.after.id END`.
- `whenMatchedDelete` deve vir **antes** de `whenMatchedUpdate`.

### 26.4 Duplicatas na Silver

- MySQL: confirme offset composto `source.file + ':' + source.pos`.
- SQL Server: confirme `source.change_lsn` (não `source.lsn`).
- Confirme `_offset_key` armazenado e usado no MERGE.

### 26.5 Schema incompatível

- Verifique política de compatibilidade.
- Adicione colunas com `DEFAULT` no banco antes de evoluir o schema.

### 26.6 Disk Full PostgreSQL

- Drope o slot ou aumente `max_slot_wal_keep_size`.
- Alertas para `lag_bytes` antes do limiar crítico.

### 26.7 OOM no Spark

- Migre para `dropDuplicatesWithinWatermark` (Spark 3.5+) com janela adequada.

---

## 27. Conclusão

CDC log-based combinado com Kafka e arquitetura em camadas (Bronze → Silver → Gold) é o padrão de referência para pipelines de dados modernos.

CDC em produção não é sobre capturar eventos — é sobre garantia de ordenação, recuperação determinística, idempotência real, monitoramento ativo e governança de schema.

Os pilares de um pipeline CDC maduro:

- **Imutabilidade da Bronze** como fonte de verdade e ponto de replay.
- **Debezium com particionamento correto** (PK como message key).
- **Merge idempotente na Silver** com `CASE WHEN` para DELETEs e deduplicação por `_offset_key`.
- **Snapshot incremental** sem lock global.
- **Schema evolution gerenciada** com Avro/Protobuf e Schema Registry.
- **Segurança desde a origem** — mascaramento no conector, crypto shredding, TLS.
- **Monitoramento proativo** — slot lag, consumer lag, alertas de schema.
- **Testes automatizados** — DELETEs, replay, schema change, resiliência.
- **HA multi-região** com MirrorMaker 2 e DR testado.

---

## Glossário

| Termo | Significado |
|-------|-------------|
| **WAL** | Write-Ahead Log (PostgreSQL) |
| **Binlog** | Binary Log (MySQL) |
| **LSN** | Log Sequence Number (PostgreSQL, SQL Server) |
| **Resume Token** | Checkpoint do MongoDB Change Streams; único mesmo após failover |
| **Replication Slot** | Mecanismo PostgreSQL que retém WAL até ser consumido |
| **Offset** | Posição no log que permite retomada exata após falha |
| **Schema Registry** | Serviço que armazena schemas Avro/JSON/Protobuf com compatibilidade |
| **PII** | Personally Identifiable Information |
| **CDC** | Change Data Capture |
| **Debezium** | Conector open source de referência para CDC log-based |
| **Kafka Connect** | Framework para conectar Kafka a sistemas externos |
| **Delta Lake / Iceberg / Hudi** | Formatos de tabela open source para data lakes com upsert/delete |
| **GTID** | Global Transaction Identifier (MySQL) |
| **Tombstone** | Mensagem com valor null publicada após DELETE para sinalizar compactação |
| **MirrorMaker 2** | Ferramenta Kafka para replicação cross-cluster |
| **dropDuplicatesWithinWatermark** | Spark 3.5+: deduplicação com janela temporal limitada |
| **Crypto shredding** | Destruir chave de criptografia torna dados históricos irrecuperáveis |
| **`delete.retention.ms`** | Tempo que tombstones são retidos em tópicos compactados (padrão: 24h) |
| **`change_lsn`** | Campo SQL Server que identifica a mudança específica (usar para deduplicação) |
| **`commit_scn` / `scn` / `xid`** | Campos Oracle para identificação única de evento (sempre usar combinados) |

---

## Referências

- [Debezium Documentation](https://debezium.io/documentation/)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Delta Lake Documentation](https://docs.delta.io/latest/index.html)
- [Apache Iceberg Documentation](https://iceberg.apache.org/docs/latest/)
- [Apache Hudi Documentation](https://hudi.apache.org/docs/overview/)
- [Confluent Schema Registry](https://docs.confluent.io/platform/current/schema-registry/index.html)
- [Testcontainers for Python](https://testcontainers-python.readthedocs.io/)
- [AWS DMS](https://aws.amazon.com/dms/)
- [Google Datastream](https://cloud.google.com/datastream)
- [Azure Data Factory](https://azure.microsoft.com/en-us/products/data-factory/)
- [Spark dropDuplicatesWithinWatermark](https://spark.apache.org/docs/3.5.0/structured-streaming-programming-guide.html)
- [Kafka MirrorMaker 2](https://kafka.apache.org/documentation/#georeplication)

---

> **Pipeline CDC maduro = log-based + Debezium 2.5+ + Kafka 3.7+ + Connect EOS v2 + Bronze imutável + `CASE WHEN` no merge + `whenMatchedDelete` primeiro + deduplicação por offset composto específico por banco + `dropDuplicatesWithinWatermark` em streams longos + crypto shredding antes da Bronze + schema evolution gerenciada com Avro/Protobuf + PII mascarado na origem + testes automatizados + monitoramento contínuo + resiliência validada em staging + HA multi-região + OPTIMIZE/ZORDER/VACUUM programados + custo controlado.**
