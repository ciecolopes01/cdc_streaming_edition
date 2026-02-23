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
- **Offset rastreado:** a posição de leitura no log é persistida, garantindo retomada sem perda além do esperado.
- **Replay confiável:** é possível reprocessar eventos históricos para reconstruir estados, alimentar novos consumidores ou corrigir erros de transformação.
- **Idempotência:** processar o mesmo evento duas vezes deve produzir o mesmo resultado que processar uma vez.
- **Baixo impacto no banco:** a captura ocorre via leitura do log, sem sobrecarga nas transações originais.
- **Observabilidade operacional:** lag, saúde do slot e estado do conector são visíveis e alertáveis.

CDC mal implementado gera duplicidade silenciosa, perda de eventos, corrupção lógica, disk full no banco e snapshots forçados inesperados.

---

## 2. Versões Recomendadas

Evite stacks legadas. CDC é extremamente sensível ao comportamento interno de WAL/binlog, e versões antigas possuem bugs conhecidos de replication que afetam a confiabilidade.

| Componente    | Versão recomendada | Observação |
|---------------|--------------------|------------|
| Kafka         | 3.7+               | 3.6 é estável; 3.7 é a minor mais adotada em 2026. `enable.idempotence=true` padrão desde 3.0 |
| Kafka Connect | 3.7+               | Worker distribuído recomendado para produção; EOS v2 requer Connect 3.3+ |
| Debezium      | 2.5+               | 2.4+: heartbeat aprimorado. 2.5+: incremental snapshot com pause/resume via signal table. API alterada em relação ao 1.x |
| PostgreSQL    | 14+                | `pgoutput` nativo; `max_slot_wal_keep_size` obrigatório em produção desde PG 13 |
| MySQL         | 8.0.34+ ou 8.4 LTS | `expire_logs_days` removido no 8.4 |
| Spark         | 3.5+               | `dropDuplicatesWithinWatermark` disponível desde 3.5 |
| Delta Lake    | 3.x                | Artefato: `delta-spark`. Liquid Clustering disponível desde 3.1 |

> ⚠ **Atenção ao migrar do Debezium 1.x para 2.x:** várias propriedades foram renomeadas. Em particular, `database.dbname` foi substituído por `database.names` e `database.history.*` por `schema.history.internal.*`. Valide todas as configurações antes da migração.

---

## 3. Quando Não Usar CDC Log-based

CDC log-based é a solução certa para a maioria dos pipelines de dados em tempo real — mas não para todos os cenários. Aplicar a receita no lugar errado gera complexidade sem benefício correspondente.

**Evite CDC log-based quando:**

- **A tabela não tem chave primária definida.** O Debezium depende da chave primária para compor a message key do Kafka e garantir ordering por entidade. Tabelas sem PK podem ser capturadas em modo `REPLICA IDENTITY FULL` no PostgreSQL, mas isso aumenta drasticamente o volume do WAL e pode degradar o banco. Em MySQL, eventos de UPDATE e DELETE em tabelas sem PK são publicados sem identificador único, tornando o merge na Silver extremamente difícil.

- **A fonte de dados não expõe o transaction log.** Serviços SaaS (Salesforce, HubSpot, Stripe, Shopify) não oferecem acesso ao log interno. Use conectores baseados em API incremental (Airbyte, Fivetran) ou webhooks nativos do serviço.

- **O volume de dados é baixo e a latência tolerada é alta.** Para tabelas pequenas com poucas escritas e SLA de horas, o custo operacional de manter Debezium + Kafka + replication slot não se justifica. Um simples job de polling com `updated_at` atende com muito menos complexidade.

- **O banco de dados não suporta log-based CDC.** SQLite, Microsoft Access e bancos embarcados não possuem mecanismo de transaction log exposto. Bancos em planos gerenciados restritivos podem não permitir as configurações necessárias (`wal_level=logical`, `binlog_format=ROW`).

- **O schema muda com altíssima frequência de forma destrutiva.** Pipelines com `DROP COLUMN` e `RENAME COLUMN` recorrentes exigem coordenação contínua entre produtor e consumidores. Considere event sourcing na camada de aplicação como alternativa.

> ℹ **O critério decisivo:** se você precisa de baixa latência, captura de DELETEs, rastreabilidade de toda a história de uma entidade e o banco suporta log-based CDC — use CDC log-based. Nos demais casos, avalie a alternativa mais simples primeiro.

---

## 4. Níveis de Maturidade do CDC

| Nível | Estratégia | Captura DELETE? | Ordem Transacional? | Impacto no Banco |
|---|---|---|---|---|
| 1 — Watermark | Coluna `updated_at` | ❌ Não | ❌ Não | Baixo |
| 2 — Change Tracking¹ | Metadados SQL Server | ✅ Sim | Parcial | Baixo |
| 3 — Trigger-based | Triggers de auditoria | ✅ Sim | Parcial | Alto |
| 4 — Log-based | WAL / Binlog / CDC nativo | ✅ Sim | ✅ Sim | Mínimo |

> ¹ Change Tracking é um recurso exclusivo do Microsoft SQL Server que registra *quais* linhas mudaram, mas não captura os valores anteriores (`before`). Não deve ser confundido com o CDC nativo (nível 4).

### 4.1 Nível 2 — Change Tracking vs CDC Nativo (SQL Server)

O Microsoft SQL Server oferece dois recursos com nomes semelhantes, mas mecanismos distintos:

- **Change Tracking (Nível 2):** registra apenas *quais* linhas foram alteradas e o tipo de operação, sem armazenar os valores anteriores. Não entrega o estado `before` — essencial para pipelines CDC completos.
- **CDC nativo (Nível 4):** lê diretamente o transaction log e captura o estado completo `before` e `after` de cada operação.

> ℹ **O Debezium para SQL Server utiliza o CDC nativo (`sys.sp_cdc_enable_table`), e não o Change Tracking. Certifique-se de habilitar o recurso correto em produção.**

### 4.2 Nível 4 — Log-based (Padrão Enterprise)

Leitura direta do transaction log: WAL no PostgreSQL, Binlog no MySQL, CDC nativo no SQL Server, Change Streams no MongoDB. Impacto mínimo nas transações originais, captura completa e suporte nativo a replay por offset.

> ℹ **Debezium** é o conector open source de referência da indústria. Suporta PostgreSQL (WAL), MySQL (Binlog), SQL Server (CDC nativo), MongoDB (Change Streams), Oracle e outros, publicando eventos diretamente em tópicos Kafka.

---

## 5. Arquitetura CDC com Kafka e Streaming

```
[ Banco de Dados: PostgreSQL / MySQL / SQL Server / MongoDB ]
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

O conector CDC lê o log do banco e publica cada mudança como um evento imutável em tópicos Kafka. Cada tabela corresponde a um tópico separado. Consumidores podem ser batch, streaming, pipelines de ML ou qualquer sistema downstream — todos completamente desacoplados entre si e do banco de origem.

### 5.1 Kafka como Backbone — Benefícios e Configurações-Chave

- **Retenção configurável:** eventos ficam disponíveis por horas, dias ou indefinidamente, permitindo que novos consumidores façam backfill completo.
- **Particionamento por chave primária:** o Debezium usa a chave primária da tabela como *message key*. Isso garante que todas as mudanças de uma mesma linha caiam sempre na mesma partição, preservando a ordem de eventos por entidade. Algumas considerações críticas para produção:
  - **Hash estável:** o Debezium usa o hash da PK serializada como message key. Nunca altere o tipo da PK ou a forma de serialização sem recalcular o mapeamento de partições — isso pode fazer eventos da mesma linha irem para partições diferentes.
  - **Hot partitions:** PKs com baixa cardinalidade (ex: enum, boolean, IDs sequenciais concentrados) podem gerar partições com volume desproporcional. Monitore o throughput por partição. Em casos extremos, considere `transforms` customizados para distribuir a carga.
  - **Chave composta:** quando a PK é composta (ex: `(tenant_id, order_id)`), o Debezium serializa todos os campos como chave. Certifique-se de que o consumer usa a chave completa para ordenação — não apenas um dos campos.
  - **Mudança de PK em produção:** alterar a PK de uma tabela que já tem CDC ativo é equivalente a reprocessamento full. O histórico de eventos com a chave antiga e os eventos com a nova chave serão tratados como entidades distintas pelo consumer. Planeje como uma migração completa, não como uma alteração de schema normal.
- **Schema Registry:** integração com Confluent Schema Registry ou AWS Glue Schema Registry para versionamento de schemas e compatibilidade evolutiva entre produtor e consumidores.
- **Idempotência do produtor:** a partir do Kafka 3.0, `enable.idempotence=true` é o padrão. Em versões anteriores, configure explicitamente para garantir que o produtor não publique duplicatas em caso de retentativa.

> ℹ **Alterar a message key sem cuidado quebra a garantia de ordering por entidade.** Customizações via `transforms` devem ser validadas cuidadosamente em ambiente de staging.

---

## 6. Banco de Origem — Detalhamento Técnico

### 6.1 PostgreSQL — Logical Decoding + WAL

#### Requisitos

```sql
-- Requer reinicialização do servidor após ALTER SYSTEM
ALTER SYSTEM SET wal_level                       = logical;
ALTER SYSTEM SET max_replication_slots            = 10;
ALTER SYSTEM SET max_wal_senders                  = 10;

-- Obrigatório em ambientes com múltiplos conectores simultâneos.
-- Sem esses parâmetros, conectores adicionais podem falhar silenciosamente
-- por esgotamento de worker processes.
ALTER SYSTEM SET max_logical_replication_workers  = 10;  -- >= max_replication_slots
ALTER SYSTEM SET max_worker_processes             = 20;  -- deve cobrir todos os workers do PG

-- Safety net obrigatório em produção (disponível desde PostgreSQL 13).
-- Limita o volume máximo de WAL retido por replication slots inativos.
-- Sem isso, um slot parado pode acumular WAL indefinidamente até disk full.
-- Ajuste conforme o espaço disponível em disco; -1 = sem limite (perigoso).
ALTER SYSTEM SET max_slot_wal_keep_size           = '20GB';
```

#### Publicação e Replication Slot

```sql
-- Opção A: publicação por tabela (mais controle, mais manutenção)
-- ATENÇÃO: novas tabelas adicionadas ao banco NÃO entram automaticamente.
-- Para cada nova tabela capturada, execute:
--   ALTER PUBLICATION dbz_publication ADD TABLE public.nova_tabela;
CREATE PUBLICATION dbz_publication FOR TABLE public.orders;

-- Opção B: publicação para todas as tabelas (mais simples, menos controle)
-- Novas tabelas entram automaticamente. Use column.include.list /
-- table.include.list no conector para filtrar o que realmente importa.
CREATE PUBLICATION dbz_publication FOR ALL TABLES;

-- Criar replication slot dedicado ao Debezium
SELECT pg_create_logical_replication_slot(
    'dbz_slot',
    'pgoutput'     -- nativo no PostgreSQL 10+; ou 'wal2json' se instalado
);

-- Monitorar o lag do slot (executar periodicamente)
SELECT slot_name,
       active,
       pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes
FROM   pg_replication_slots;
```

#### Como funciona

O Replication Slot mantém o `restart_lsn` — a posição a partir da qual o WAL deve ser retido. Enquanto o conector estiver ativo e consumindo, esse ponteiro avança e o WAL é reciclado normalmente.

> ✗ **Risco crítico — Disk Full:** se o conector parar, o `restart_lsn` não avança. O WAL acumula indefinidamente, podendo causar queda total do banco. Configure alertas quando `lag_bytes` ultrapassar o threshold definido. Se o conector ficar parado por longo período, avalie dropar o slot manualmente e recriar via snapshot incremental do Debezium:
> ```sql
> SELECT pg_drop_replication_slot('dbz_slot');
> ```

> ⚠ **PostgreSQL 13+:** configure `max_slot_wal_keep_size` como safety net para limitar o crescimento máximo de WAL retido por slots, evitando disk full em casos de conector inativo.

---

### 6.2 MySQL / MariaDB — Binlog ROW Format

```ini
# ── MySQL 5.7 ──────────────────────────────────────────────────────────
[mysqld]
server-id        = 1           # Deve ser ÚNICO em toda a topologia (replicação, cluster).
                               # Conflito de server-id causa comportamento indefinido na replicação.
log_bin          = ON          # Habilita o Binlog
binlog_format    = ROW         # Obrigatório para CDC
binlog_row_image = FULL        # Garante before/after completos
expire_logs_days = 7           # Retenção (deprecated desde MySQL 8.0)

# ── MySQL 8.0+ ──────────────────────────────────────────────────────────
# expire_logs_days está deprecated desde 8.0 e REMOVIDO no MySQL 8.4
binlog_expire_logs_seconds = 604800   # 7 dias em segundos
```

> ⚠ **MySQL 8.4 LTS:** `expire_logs_days` foi removido na especificação. Dependendo da build/distribuição, mantê-lo configurado pode impedir a inicialização do servidor ou ser silenciosamente ignorado com um warning — não há comportamento garantido. A prática segura é removê-lo e usar exclusivamente `binlog_expire_logs_seconds`. O valor deve ser maior que o tempo máximo de inatividade tolerado para o conector — se ele ficar parado além desse período, os logs serão descartados e um novo snapshot completo será necessário.

#### Cálculo estratégico da retenção de binlog

A retenção não deve ser um valor arbitrário. Calcule com base em três variáveis:

```
retenção mínima (segundos) =
    tempo_máximo_inatividade_do_connector   (ex: 4h = 14400s para manutenção)
  + janela_de_SLA_de_recuperação            (ex: 2h = 7200s para identificar e agir)
  + margem_de_segurança                     (ex: 50% do total acima)
```

**Exemplo:** para um connector com manutenção de até 4h e SLA de recuperação de 2h:
- Mínimo: 14400 + 7200 = 21600s (6h)
- Com margem de 50%: 32400s (~9h)
- Recomendado configurar: `binlog_expire_logs_seconds = 604800` (7 dias) como piso seguro para a maioria dos cenários de produção.

Em ambientes com **alto throughput de escritas**, o binlog cresce rapidamente. Monitore o espaço ocupado (`SHOW BINARY LOGS;`) e ajuste a retenção considerando o espaço em disco disponível — retenção longa em banco com alto volume pode consumir centenas de GBs.

#### GTID — Cuidados Essenciais

Se GTID (`gtid_mode=ON`) estiver habilitado:

- **Nunca manipule `gtid_purged` manualmente** após o conector estar configurado. Isso pode tornar a recuperação impossível, pois o Debezium não conseguirá localizar o ponto de retomada no binlog.
- **Nunca force purge de binlogs** sem antes validar que o offset atual do conector foi consumido.
- Em caso de failover para réplica, o GTID garante consistência de posição — mas valide que a réplica está em dia antes de redirecionar o conector.

O offset do Debezium para MySQL é composto pelo nome do arquivo de Binlog mais a posição dentro dele (ex: `mysql-bin.000123:4567`). Em ambientes GTID, o conector também persiste o GTID set processado.

---

### 6.3 SQL Server — CDC Nativo

```sql
-- 1. Habilitar CDC no banco de dados
EXEC sys.sp_cdc_enable_db;

-- 2. Habilitar CDC na tabela alvo (CDC nativo, não Change Tracking)
--    ATENÇÃO: CDC deve ser habilitado individualmente por tabela.
--    Habilitar no banco não habilita automaticamente todas as tabelas.
EXEC sys.sp_cdc_enable_table
    @source_schema     = 'dbo',
    @source_name       = 'orders',
    @role_name         = NULL,
    @supports_net_changes = 0;  -- Debezium usa "all changes"; net changes não é necessário

-- 3. Verificar que a tabela está com CDC habilitado
SELECT name, is_cdc_enabled
FROM   sys.tables
WHERE  name = 'orders';

-- 4. Verificar instâncias de captura criadas
SELECT * FROM cdc.change_tables;
```

> ℹ O parâmetro `@supports_net_changes = 1` habilita funções que retornam apenas o estado final após múltiplas alterações. O Debezium utiliza as funções de "all changes" (`cdc.fn_cdc_get_all_changes_...`), portanto `supports_net_changes` pode ficar como 0 sem impacto.

> ⚠ **Configure a retenção do CDC no SQL Server** via `cdc.cleanup_change_table`. O padrão é 3 dias — insuficiente para cenários com janelas de manutenção longas ou conectores pausados. Ajuste conforme o SLA de recuperação:
> ```sql
> -- Alterar retenção para 7 dias (em minutos)
> EXEC sys.sp_cdc_change_job @job_type = 'cleanup', @retention = 10080;
> ```

> ⚠ **SQL Server e Log Shipping:** se o servidor usa Log Shipping para replicação para réplicas secundárias, habilitar CDC pode aumentar o volume do transaction log de forma significativa, pois o SQL Agent Job de captura lê o log continuamente. Monitore o crescimento do log e o impacto no Log Shipping antes de habilitar CDC em produção com alta taxa de escritas.

---

### 6.4 MongoDB — Change Streams

```javascript
// Escutar mudanças na coleção orders
const changeStream = db.orders.watch([
    { $match: { operationType: { $in: ['insert', 'update', 'delete'] } } }
], {
    fullDocument: 'updateLookup',  // inclui documento completo após update
    resumeAfter: savedResumeToken  // retoma do checkpoint salvo
});

changeStream.on('change', (event) => {
    // IMPORTANTE: salvar o resume token como checkpoint após cada evento
    const checkpoint = event._id;
    processEvent(event.operationType, event.fullDocument, checkpoint);
});
```

> ℹ **O MongoDB exige replica set ativo** para habilitar Change Streams, mesmo em instâncias únicas de desenvolvimento. Use `rs.initiate()` antes de configurar o conector.

> ⚠ **Resume Token é opaco e deve ser persistido integralmente.** Não tente parsear, modificar ou reconstruir o Resume Token — sua estrutura interna é privada e pode mudar entre versões do MongoDB. Sempre persista a string completa retornada em `event._id`. O campo `clusterTime` presente no evento **não garante unicidade** — dois eventos diferentes podem ter o mesmo `clusterTime`. O Resume Token é a única referência segura de checkpoint e de retomada após failover de replica set.

---

## 7. Debezium — Configuração Correta

### 7.1 PostgreSQL

```json
{
  "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
  "database.hostname": "postgres",
  "database.port": "5432",
  "database.user": "debezium",
  "database.password": "${DB_PASSWORD}",
  "database.dbname": "appdb",
  "plugin.name": "pgoutput",
  "publication.name": "dbz_publication",
  "slot.name": "dbz_slot",
  "snapshot.mode": "initial",
  "tombstones.on.delete": "true",
  "topic.prefix": "dbserver1"
}
```

### 7.2 MySQL / SQL Server (Debezium 2.x)

> ⚠ **Debezium 2.x:** a propriedade `database.dbname` (singular) foi substituída por `database.names` (plural) nos conectores MySQL e SQL Server, pois esses conectores suportam múltiplos bancos em uma única instância. Usar `database.dbname` em Debezium 2.x pode causar falha silenciosa de configuração.

```json
{
  "connector.class": "io.debezium.connector.sqlserver.SqlServerConnector",
  "database.hostname": "sqlserver",
  "database.port": "1433",
  "database.user": "debezium",
  "database.password": "${DB_PASSWORD}",
  "database.names": "appdb",
  "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
  "schema.history.internal.kafka.topic": "schema-changes.orders",
  "table.include.list": "dbo.orders",
  "topic.prefix": "dbserver1",
  "tombstones.on.delete": "true"
}
```

> ℹ **`tombstones.on.delete=true`** publica um evento tombstone (valor `null`) após cada DELETE, sinalizando ao compactador do Kafka que pode remover a entrada do log compactado. Mantenha habilitado para tópicos com `cleanup.policy=compact`.

### 7.3 Deploy via API REST do Kafka Connect

```bash
curl -X POST http://kafka-connect:8083/connectors \
  -H 'Content-Type: application/json' \
  -d @debezium-connector-config.json

# Verificar status do conector
curl http://kafka-connect:8083/connectors/my-connector/status
```

---

## 8. Formato de Evento no Tópico Kafka

O Debezium padroniza o envelope de evento para todos os bancos suportados. O payload completo de um UPDATE contém:

```json
{
  "payload": {
    "before": {
      "id": 42,
      "status": "open",
      "total": 150.00
    },
    "after": {
      "id": 42,
      "status": "paid",
      "total": 150.00
    },
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

> ⚠ **Os campos dentro de `source` variam por conector** — o exemplo acima é do PostgreSQL. Cada banco expõe campos distintos:
>
> | Banco | Campo de posição | Campo de transação |
> |---|---|---|
> | PostgreSQL | `lsn` (inteiro) | `txId` |
> | MySQL / MariaDB | `file` + `pos` (composto) | `gtid` (se GTID ativo) |
> | SQL Server | `change_lsn` + `commit_lsn` | — |
> | MongoDB | `resume_token` (string BSON) | `clusterTime` (não único) |
>
> Qualquer lógica de deduplicação ou chave de offset deve considerar qual banco está sendo capturado — não existe um campo universal.

**Valores de `op`:** `c` = insert · `u` = update · `d` = delete · `r` = snapshot/read

**Regra de ouro:** `after` é `null` para DELETE e `before` é `null` para INSERT. Qualquer código que acesse `b.after.id` sem verificar o tipo de operação falhará silenciosamente em eventos de DELETE.

> ✗ **Atenção crítica:** nunca acesse diretamente `b.after.id` em merges — use `CASE WHEN b.op = 'd' THEN b.before.id ELSE b.after.id END`. Detalhado na seção 10.1.

---

## 9. Camada Bronze — A Fundação Imutável

A camada Bronze armazena eventos exatamente como chegaram do stream, sem transformação, deduplicação ou merge. Essa imutabilidade é o pilar que torna o pipeline auditável e reprocessável a qualquer momento.

Cada registro na Bronze deve conter obrigatoriamente:

- **Operação (`op`):** `c` (insert), `u` (update), `d` (delete) ou `r` (snapshot/read).
- **Estado `before`:** imagem da linha antes da operação. É `null` para INSERTs.
- **Estado `after`:** imagem da linha após a operação. É `null` para DELETEs.
- **Offset:** posição exata do evento no transaction log (LSN no PostgreSQL/SQL Server, `file+position` no MySQL, resume token no MongoDB).
- **Timestamp de origem (`source.ts_ms`):** momento do commit no banco de origem, não o momento de publicação no Kafka.
- **Metadados do conector:** versão, nome do servidor, transaction ID e nome da tabela.

> ℹ **Nunca sobrescreva dados na Bronze.** Em reprocessamentos, adicione nova partição com sufixo de versão. A imutabilidade é o que garante o replay total.

---

## 10. Camadas Silver e Gold

### 10.1 Silver — Merge Incremental Correto e Idempotente

A Silver aplica a semântica transacional sobre os eventos crus da Bronze, produzindo uma visão consistente e atualizada de cada entidade.

**✗ Bug comum — acesso direto a `after.id`:**

```python
# ❌ INCORRETO — falha silenciosamente em eventos de DELETE
# Em DELETEs, after = null, portanto b.after.id = null
# A condição nunca fará match e o DELETE nunca será aplicado na Silver
.merge(
    batch_df.alias('b'),
    condition='s.id = b.after.id'   # ERRADO: after é null em DELETEs
)
```

**Solução correta — CASE WHEN por tipo de operação, com deduplicação robusta por banco:**

```python
from delta.tables import DeltaTable
from pyspark.sql import functions as F


def build_offset_key(df, source_type: str):
    """
    Constrói chave de deduplicação conforme o banco de origem.
    PostgreSQL / SQL Server : source.lsn  (inteiro único por banco)
    MySQL / MariaDB         : source.file + ':' + source.pos  (composto — pos reinicia por arquivo)
    MongoDB                 : source.resume_token  (string completa; clusterTime não garante unicidade)
    """
    if source_type in ('postgresql', 'sqlserver'):
        return df.withColumn('_offset_key', F.col('source.lsn').cast('string'))
    elif source_type in ('mysql', 'mariadb'):
        return df.withColumn(
            '_offset_key',
            F.concat_ws(':', F.col('source.file'), F.col('source.pos').cast('string'))
        )
    elif source_type == 'mongodb':
        return df.withColumn('_offset_key', F.col('source.resume_token'))
    else:
        raise ValueError(f"source_type não suportado: {source_type}")


def upsert_to_silver(batch_df, batch_id, source_type='postgresql'):
    silver = DeltaTable.forPath(spark, '/silver/orders')

    # 1. Adicionar chave de offset conforme o banco de origem
    deduped = build_offset_key(batch_df, source_type)

    # 2. Deduplicação dentro do batch.
    #    Em at-least-once, o mesmo offset pode chegar mais de uma vez após falha.
    #
    #    ATENÇÃO — dropDuplicates mantém o PRIMEIRO registro com aquela chave,
    #    não o último. Em foreachBatch (modo micro-batch), a ordem dentro do batch
    #    não é determinística. Se precisar garantir "o mais recente", ordene por
    #    source.ts_ms ANTES de chamar dropDuplicates.
    #
    #    ATENÇÃO — em Structured Streaming contínuo (fora de foreachBatch),
    #    dropDuplicates acumula estado indefinidamente. Para streams de longa
    #    duração, use dropDuplicatesWithinWatermark (Spark 3.5+) com uma janela
    #    adequada ao SLA de reprocessamento:
    #
    #      deduped = (
    #          deduped
    #          .withWatermark('event_time', '2 hours')
    #          .dropDuplicatesWithinWatermark(['_offset_key'])
    #      )
    #
    #    Sem watermark, o estado cresce sem limite e causa OOM em streams longos.
    #
    #    EVITE chaves compostas genéricas como concat(lsn, txId, op):
    #    - txId pode se repetir entre bancos diferentes
    #    - 'op' não garante unicidade (dois updates na mesma txId têm LSN diferente,
    #       mas se você incluir op e txId sem lsn, pode colidir)
    #    - Em MySQL não existe 'lsn'; a chave deve ser file+pos
    #    Use a função build_offset_key abaixo, que é específica por banco.
    deduped = deduped.dropDuplicates(['_offset_key'])

    # 3. MERGE com ordem correta das cláusulas:
    #    whenMatchedDelete ANTES de whenMatchedUpdate garante que um evento DELETE
    #    seja aplicado como deleção, sem risco de cair acidentalmente na cláusula
    #    de UPDATE por edge case de condição.
    silver.alias('s').merge(
        deduped.alias('b'),
        # ✅ CORRETO: usa before.id para DELETEs (after é null nesses eventos)
        condition='''
            s.id = CASE
                WHEN b.op = 'd' THEN b.before.id
                ELSE b.after.id
            END
        '''
    ).whenMatchedDelete(
        condition="b.op = 'd'"                        # ← DELETE primeiro
    ).whenMatchedUpdate(
        # Inclui 'c' (INSERT) para idempotência: se um evento INSERT chegar
        # para uma linha já existente (replay), ele é tratado como UPDATE
        # em vez de gerar erro de chave duplicada.
        condition="b.op IN ('u', 'c')",               # ← UPDATE depois
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

> ⚠ **A chave de offset no MySQL é composta (`source.file + ':' + source.pos`)**, não apenas o `pos`. Usar somente `source.pos` como chave de deduplicação gera falsos positivos quando um novo arquivo de Binlog é criado — o `pos` reinicia do zero.

> ⚠ **Para MongoDB, o campo correto é `source.resume_token`**, não `source.ord`. O campo `ord` é um contador sequencial que não garante unicidade entre sessões ou após failover.

### 10.2 Gold — Modelo Analítico Pronto para Consumo

A Gold entrega agregações, dimensões e fatos modelados para consumo por ferramentas de BI, APIs ou modelos de ML. Pode ser reconstruída a qualquer momento a partir da Bronze, graças à imutabilidade do histórico.

- **Modelos dimensionais:** Star Schema ou Snowflake Schema para ferramentas como Power BI, Tableau e Looker.
- **Feature stores:** tabelas atualizadas em near real-time para alimentar modelos de ML.
- **APIs de leitura:** dados servidos com latência de segundos, construídos sobre a Gold com SLA definido.

---

## 11. Semânticas de Entrega e Idempotência

### 11.1 At-Least-Once — Padrão Debezium

O Debezium garante *at-least-once delivery*: em caso de falha e retomada, alguns eventos podem ser republicados. Consumidores devem ser idempotentes — processar o mesmo evento duas vezes deve produzir o mesmo resultado que processar uma vez.

Na prática, idempotência na Silver é implementada via:

- **Deduplicação por offset:** use a `_offset_key` adequada para o banco conforme demonstrado na seção 10.1.
- **MERGE com chave primária:** a operação de MERGE é naturalmente idempotente para UPDATEs e INSERTs.
- **Tombstone para DELETEs:** o Debezium publica dois eventos para um DELETE: o evento com `op='d'` e um tombstone (mensagem com valor `null`). Em tópicos com `cleanup.policy=compact`, o compactador **não remove a tombstone imediatamente** — ela fica retida por `delete.retention.ms` (padrão: 24 horas) para que consumidores que estejam temporariamente offline possam recebê-la antes da remoção definitiva. Consumidores devem tratar tombstones explicitamente (ignorar ou processar), não apenas ignorar mensagens `null`.

### 11.2 Exactly-Once com Kafka Transactions

Exactly-once **não significa deduplicação automática nem impossibilidade de reprocessamento**. Significa que a transação entre produção e commit de offset é atômica — mas isso só vale quando todas as condições abaixo são satisfeitas simultaneamente:

- Produtor transacional no Kafka (`enable.idempotence=true`, padrão desde Kafka 3.0)
- Consumidor com `isolation.level=read_committed` — sem isso, o consumidor pode ler mensagens de transações ainda não commitadas (uncommitted reads)
- Kafka Connect em modo exactly-once (configurado explicitamente; o padrão do Connect é at-least-once)
- Sink idempotente — o destino (Delta Lake, banco, etc.) deve lidar corretamente com retentativas

Se qualquer uma dessas condições não for atendida, a semântica real é at-least-once, independentemente da configuração do produtor.

> ℹ **Kafka Connect — EOS v2:** o Connect suporta exactly-once para source connectors a partir da versão 3.3+, via `exactly.once.source.support=enabled` no worker. **O Debezium como source não garante exactly-once por padrão** — a garantia nativa é at-least-once. Para EOS end-to-end com Debezium é necessário: Connect 3.3+ com EOS v2 habilitado + sink idempotente + consumer com `isolation.level=read_committed`.
> ```properties
> # kafka-connect worker.properties
> exactly.once.source.support=enabled
> ```

> ℹ **Apache Flink com Kafka source e Delta Lake sink** oferece exactly-once nativo via two-phase commit. Adicione `io.delta:delta-flink` (Delta Lake 3.x) e habilite checkpointing do Flink no modo `EXACTLY_ONCE`.

> ℹ **Kafka Streams:** utilize `processing.guarantee=exactly_once_v2` (disponível a partir do **Kafka 2.6**) para exactly-once com melhor throughput do que o `exactly_once` legado.

---

## 12. Snapshot Inicial — Consistência e Estratégias

### 12.1 O Problema de Consistência

Ao conectar o Debezium a uma tabela existente com dados históricos, ele precisa realizar um snapshot inicial antes de começar a capturar eventos do log. O desafio é garantir que não haja lacuna nem duplicação entre o snapshot e os eventos de log que ocorrem durante a sua execução.

O snapshot não é gratuito: causa leitura massiva, pressão em I/O e aumento de latência percebida. Em produção grande, prefira snapshot incremental ou `schema_only` com backfill separado.

> ⚠ **`snapshot.mode=schema_only` — perda de dados históricos:** este modo captura apenas o schema da tabela e inicia a captura de eventos a partir do momento da conexão. **Todos os dados existentes na tabela no momento da configuração são ignorados.** Se você precisa que a Silver reflita o estado atual da tabela, é necessário realizar um backfill separado (ex: exportação via `COPY TO` / `mysqldump` importada diretamente na Bronze) e somente então ativar o conector em modo `schema_only`. Usar este modo sem backfill resulta em Silver incompleta — sem erro visível, com dados silenciosamente faltando.

### 12.2 Snapshot Tradicional (com Lock)

No modo padrão, o Debezium adquire um lock de leitura durante o snapshot para garantir que o LSN de início seja coerente com os dados lidos. Após o snapshot, o conector retoma a partir desse LSN, garantindo zero gap.

> ⚠ **O lock de snapshot pode causar bloqueio temporário** em tabelas de alta concorrência. Em produção crítica, prefira o snapshot incremental.

### 12.3 Snapshot Incremental (Chunk-based — Recomendado)

Disponível a partir do Debezium 1.6, o snapshot incremental elimina a necessidade de lock global. A tabela é lida em chunks por faixa de chave primária, e o mecanismo de watermark garante que eventos de log recebidos durante o snapshot sejam mesclados corretamente.

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

> ℹ **O snapshot incremental é seguro para produção:** não bloqueia a tabela, pode ser pausado e retomado, e garante consistência via watermark interno.

### 12.4 Snapshot via Exportação Paralela

Para tabelas muito grandes (centenas de GBs), realize o snapshot via exportação nativa (`COPY TO` no PostgreSQL, `mysqldump` no MySQL) e importe diretamente na Bronze, registrando o LSN antes da exportação. O Debezium retoma a partir desse LSN, evitando a leitura completa via JDBC.

---

## 13. Schema Evolution — DDL no Banco de Origem

Mudanças de DDL são um dos pontos mais críticos e menos documentados em pipelines CDC. Em produção, schema evolution mal gerenciada causa mais incidentes do que duplicidade de eventos. O comportamento varia conforme o banco e o tipo de mudança:

| Operação DDL | PostgreSQL | MySQL | SQL Server |
|---|---|---|---|
| `ADD COLUMN` | ✅ Detectado automaticamente | ✅ Detectado automaticamente | ✅ Detectado automaticamente |
| `DROP COLUMN` | ✅ Detectado | ✅ Detectado | ✅ Detectado |
| `RENAME COLUMN` | ⚠ Pode exigir reset do slot | ✅ Detectado | ⚠ Exige atenção |
| `CHANGE TYPE` | ⚠ Pode quebrar consumidores | ⚠ Pode quebrar consumidores | ⚠ Pode quebrar consumidores |

### 13.1 Formato de Serialização — Avro vs JSON vs Protobuf

A escolha do formato de serialização afeta diretamente o comportamento em schema evolution:

| Formato | Schema enforcement | Evolução compatível | Overhead | Indicado para |
|---|---|---|---|---|
| **Avro** | Forte (via Registry) | BACKWARD/FORWARD/FULL nativo | Baixo | Produção: maior ecossistema, melhor integração com Kafka |
| **Protobuf** | Forte (via Registry) | Field numbers garantem compatibilidade | Baixo | Produção: grpc-friendly, ótimo para cross-language |
| **JSON Schema** | Moderado (via Registry) | Possível, mas menos rigoroso | Alto | Desenvolvimento ou integração com sistemas JSON-nativos |
| **JSON sem Registry** | Nenhum | Sem garantia | Alto | ❌ Não recomendado para produção |

> ⚠ **Avro e campos nullable:** em Avro, um campo só pode ser `null` se o tipo for declarado como union `["null", "tipo"]`. Ao adicionar uma coluna nullable no banco, o schema Avro **deve** declará-la como union, caso contrário eventos com valor `null` falharão na deserialização. Sempre declare campos novos como `["null", "tipo"]` com default `null` em Avro.

### 13.2 Política de Compatibilidade no Schema Registry

- **`BACKWARD` (recomendado como ponto de partida):** novos schemas podem ler dados escritos com schemas anteriores. Operações permitidas: adicionar campo com `default`, remover campo opcional.
- **`FORWARD`:** schemas antigos podem ler dados escritos com novos schemas. Permite adicionar campos opcionais.
- **`FULL`:** combinação de BACKWARD e FORWARD. Mais restritiva, maior segurança em ambientes multi-consumidor com versões diferentes em produção simultânea.
- **`NONE`:** sem validação. Apenas para desenvolvimento — nunca use em produção.

```json
// Configurar política no Schema Registry via API REST
PUT /config/dbserver1.public.orders-value
{
  "compatibility": "BACKWARD"
}
```

#### Regras práticas por operação

| Operação | Compatível com BACKWARD? | Como fazer de forma segura |
|---|---|---|
| `ADD COLUMN NOT NULL` sem default | ❌ Não | Adicionar com `DEFAULT` no banco primeiro |
| `ADD COLUMN NULL` ou com `DEFAULT` | ✅ Sim | Padrão seguro |
| `DROP COLUMN` obrigatório | ❌ Não | Deprecar → migrar consumidores → remover |
| `RENAME COLUMN` | ❌ Não | Adicionar nova coluna, migrar, remover antiga |
| `CHANGE TYPE` compatível (ex: INT→BIGINT) | ⚠ Depende | Validar no Registry antes de aplicar |
| `CHANGE TYPE` incompatível (ex: VARCHAR→INT) | ❌ Não | Requer novo tópico + migração completa |

### 13.3 Ciclo de Vida Seguro de uma Mudança de Schema

```
1. Adicionar coluna com DEFAULT no banco de origem
   ALTER TABLE orders ADD COLUMN discount DECIMAL(10,2) DEFAULT 0.0;

2. Aguardar o Debezium detectar a mudança (eventos seguintes já incluirão o novo campo)

3. Verificar no Schema Registry que o novo schema foi registrado e aprovado
   GET /subjects/dbserver1.public.orders-value/versions/latest

4. Atualizar consumidores para lidar com o novo campo (BACKWARD garante que
   consumidores antigos ainda funcionem durante a transição)

5. Após todos os consumidores atualizados, remover suporte ao schema antigo se necessário
```

> ⚠ **`DROP COLUMN` e `RENAME COLUMN` são operações destrutivas** — nenhuma política de compatibilidade permite essas mudanças sem coordenação explícita. Processo correto: (1) adicionar nova coluna e populá-la; (2) migrar todos os consumidores; (3) só então remover ou renomear a coluna antiga, preferencialmente com janela de manutenção.

> ⚠ **Alteração de tipo incompatível (ex: `VARCHAR(50)` → `TEXT`, ou `DECIMAL` → `BIGINT`):** mesmo que pareça "compatível" no banco, o schema Avro/Protobuf gerado pelo Debezium pode ser incompatível com o schema anterior. Sempre valide no Schema Registry com uma chamada de compatibilidade antes de aplicar em produção:
> ```bash
> # Testar compatibilidade antes de registrar
> POST /compatibility/subjects/dbserver1.public.orders-value/versions/latest
> {"schema": "...novo schema Avro..."}
> # Response esperado: {"is_compatible": true}
> ```

---

## 14. Segurança, PII e Conformidade

Em pipelines CDC, dados sensíveis do banco chegam integralmente à Bronze por padrão — incluindo CPF, e-mail, dados de cartão de crédito. Sem controles explícitos, o log de eventos se torna um vetor de vazamento de PII, com implicações em LGPD, GDPR e PCI-DSS.

### 14.1 Mascaramento de Colunas PII no Conector

O Debezium oferece propriedades nativas para mascarar, hashear ou excluir colunas **antes** da publicação no Kafka:

```json
{
  "name": "my-connector",
  "config": {
    "connector.class": "io.debezium.connector.sqlserver.SqlServerConnector",
    "database.hostname": "...",
    "database.port": "1433",
    "database.user": "...",
    "database.password": "...",
    "database.names": "appdb",
    "table.include.list": "dbo.orders",
    "topic.prefix": "dbserver1",
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
    "schema.history.internal.kafka.topic": "schema-changes.orders",

    "column.mask.hash.SHA-256.with.salt.mySalt": "dbo.customers.email,dbo.customers.cpf",
    "column.mask.with.12.chars": "dbo.payments.card_number,dbo.payments.cvv",
    "column.exclude.list": "dbo.customers.raw_password"
  }
}
```

> ✗ **Nunca confie apenas no controle de acesso ao tópico Kafka para proteção de PII.** O mascaramento deve ocorrer no conector — antes da publicação — para garantir que dados sensíveis não apareçam em snapshots, no log do Kafka ou em backups.

### 14.2 Segurança em Trânsito e em Repouso

```json
{
  "database.sslmode": "verify-full",
  "database.sslcert": "/secrets/client.crt",
  "database.sslkey": "/secrets/client.key",
  "database.sslrootcert": "/secrets/ca.crt"
}
```

- **TLS entre conector e banco:** configure `sslmode=verify-full` (PostgreSQL) ou `useSSL=true&requireSSL=true` (MySQL).
- **TLS entre conector e Kafka:** configure `security.protocol=SSL` e os respectivos keystores no Kafka Connect worker.
- **Criptografia em repouso na Bronze:** SSE-S3, ADLS encryption ou Parquet column encryption.
- **Rotação de credenciais:** armazene senhas em secret manager (AWS Secrets Manager, HashiCorp Vault, Azure Key Vault). Nunca faça hard-code em arquivos de configuração.

### 14.3 Controle de Acesso a Tópicos Kafka

```bash
# Apenas o conector Debezium pode produzir no tópico
kafka-acls --add \
  --allow-principal User:debezium-connector \
  --operation Write \
  --topic dbserver1.public.orders

# Apenas consumidores autorizados podem ler
kafka-acls --add \
  --allow-principal User:silver-pipeline \
  --operation Read \
  --topic dbserver1.public.orders \
  --group silver-consumer-group
```

> ℹ **Aplique o princípio do menor privilégio:** o usuário de banco no Debezium deve ter apenas leitura do log de replicação, acesso às tabelas alvo e, no PostgreSQL, o papel `REPLICATION`. Nunca use credenciais de administrador.

### 14.4 Direito ao Esquecimento (LGPD / GDPR)

A Bronze é imutável por design — o que cria um desafio específico para requisições de exclusão. As estratégias para atender ao direito ao esquecimento sem comprometer o pipeline:

- **Crypto shredding:** criptografe os dados PII com uma chave por usuário **antes de gravar na Bronze** (no conector ou via SMT, nunca após a escrita). Quando o usuário solicitar exclusão, destrua a chave — os eventos históricos permanecem, mas os dados se tornam irrecuperáveis.
- **Tombstone + reprocessamento:** publique um evento de exclusão com os campos PII nulos ou tokenizados e reconstrua a Silver a partir desse ponto.
- **Segregação de PII:** armazene PII em tabela separada referenciada por token na Bronze. A exclusão do mapeamento token → PII atende ao direito ao esquecimento sem alterar os eventos históricos.

---

## 15. Modos de Deploy do Debezium

### 15.1 Debezium Embedded no Kafka Connect (Padrão)

O modo mais utilizado em produção. O Debezium roda como um conector dentro de um cluster Kafka Connect, beneficiando-se do gerenciamento de offsets, distribuição de carga e monitoramento nativos do Connect.

**Indicado para:** times com Kafka Connect já operacional, múltiplos conectores, necessidade de escalabilidade horizontal.

### 15.2 Debezium Server (Standalone)

O **Debezium Server** é uma aplicação standalone que publica eventos diretamente em diferentes destinos sem passar pelo Kafka.

```properties
# application.properties — Debezium Server
debezium.source.connector.class=io.debezium.connector.postgresql.PostgresConnector
debezium.source.database.hostname=postgres
debezium.source.database.port=5432
debezium.source.database.user=debezium
debezium.source.database.password=${DB_PASSWORD}
debezium.source.database.dbname=ecommerce
debezium.source.table.include.list=public.orders

# Destino: Amazon Kinesis (sem necessidade de Kafka)
debezium.sink.type=kinesis
debezium.sink.kinesis.region=us-east-1
debezium.sink.kinesis.stream.name=cdc-orders
```

| Destino | Tipo | Caso de Uso |
|---|---|---|
| Kafka | Streaming | Pipeline completo com Kafka |
| Amazon Kinesis | Streaming gerenciado | Ambientes AWS sem Kafka |
| Google Pub/Sub | Streaming gerenciado | Ambientes GCP |
| Azure Event Hubs | Streaming gerenciado | Ambientes Azure |
| Redis Streams | In-memory | Latência ultra-baixa |
| HTTP/Webhook | Genérico | Integrações customizadas |
| RabbitMQ | Message broker | Ambientes com RabbitMQ |

**Indicado para:** times que não querem operar Kafka, ambientes serverless ou destinos cloud nativos.

### 15.3 Debezium Embedded Library

Para CDC dentro de uma aplicação JVM existente:

```java
DebeziumEngine<ChangeEvent<String, String>> engine = DebeziumEngine
    .create(Json.class)
    .using(props)
    .notifying(record -> processChangeEvent(record))
    .build();

ExecutorService executor = Executors.newSingleThreadExecutor();
executor.execute(engine);
```

**Indicado para:** microserviços que precisam reagir a mudanças do próprio banco sem infraestrutura de streaming (cache invalidation, event sourcing local).

---

## 16. Testando Pipelines CDC

Pipelines CDC são notoriamente difíceis de testar porque envolvem múltiplos sistemas externos e dependem do comportamento de transaction logs. Sem uma estratégia de testes, regressões silenciosas passam para produção sem detecção.

### 16.1 Testes de Unidade — Validação do Schema de Eventos

```python
import pytest
from pyspark.sql import SparkSession

@pytest.fixture(scope='session')
def spark():
    # Delta Lake 3.x: artefato correto é 'delta-spark', não 'delta-core'
    return SparkSession.builder \
        .config('spark.jars.packages', 'io.delta:delta-spark_2.12:3.2.0') \
        .config('spark.sql.extensions', 'io.delta.sql.DeltaSparkSessionExtension') \
        .config('spark.sql.catalog.spark_catalog', 'org.apache.spark.sql.delta.catalog.DeltaCatalog') \
        .getOrCreate()

# Payload simulado de um evento DELETE do Debezium
DELETE_EVENT = {
    'op': 'd',
    'before': {'id': 42, 'status': 'paid', 'total': 150.0},
    'after': None,                     # after é null em DELETEs — caso crítico
    'source': {'lsn': 99999, 'ts_ms': 1710000000000}
}

INSERT_EVENT = {
    'op': 'c',
    'before': None,                    # before é null em INSERTs
    'after': {'id': 43, 'status': 'open', 'total': 200.0},
    'source': {'lsn': 100000, 'ts_ms': 1710000001000}
}

def test_delete_event_id_extraction(spark):
    """Garante que a extração de ID usa before.id para DELETEs."""
    df = spark.createDataFrame([DELETE_EVENT])
    result = df.selectExpr(
        "CASE WHEN op = 'd' THEN before.id ELSE after.id END AS entity_id"
    )
    assert result.first()['entity_id'] == 42

def test_insert_event_id_extraction(spark):
    """Garante que a extração de ID usa after.id para INSERTs."""
    df = spark.createDataFrame([INSERT_EVENT])
    result = df.selectExpr(
        "CASE WHEN op = 'd' THEN before.id ELSE after.id END AS entity_id"
    )
    assert result.first()['entity_id'] == 43
```

### 16.2 Testes de Integração com Testcontainers

```python
import pytest
import json
import psycopg2
from kafka import KafkaConsumer
from testcontainers.postgres import PostgresContainer
from testcontainers.kafka import KafkaContainer

@pytest.fixture(scope='module')
def postgres():
    with PostgresContainer('postgres:15') \
            .with_env('POSTGRES_DB', 'testdb') \
            .with_command('postgres -c wal_level=logical') as pg:
        yield pg

@pytest.fixture(scope='module')
def kafka():
    with KafkaContainer('confluentinc/cp-kafka:7.5.0') as k:
        yield k

@pytest.fixture
def kafka_consumer(kafka):
    """Fixture reutilizável com timeout para evitar travamento em caso de falha."""
    consumer = KafkaConsumer(
        'dbserver1.public.orders',
        bootstrap_servers=kafka.get_bootstrap_server(),
        value_deserializer=lambda m: json.loads(m.decode('utf-8')),
        auto_offset_reset='earliest',
        consumer_timeout_ms=10000   # falha o teste se nenhuma mensagem chegar em 10s
    )
    yield consumer
    consumer.close()

def test_insert_propagates_to_kafka(postgres, kafka_consumer):
    """Testa que um INSERT no banco gera o evento correto no tópico Kafka."""
    conn = psycopg2.connect(postgres.get_connection_url())
    cur = conn.cursor()
    cur.execute("INSERT INTO orders (id, status) VALUES (1, 'open')")
    conn.commit()
    cur.close()
    conn.close()

    message = next(kafka_consumer)
    payload = message.value['payload']

    assert payload['op'] == 'c'
    assert payload['after']['id'] == 1
    assert payload['after']['status'] == 'open'
    assert payload['before'] is None

def test_delete_event_uses_before_id(postgres, kafka_consumer):
    """
    Garante que eventos de DELETE contêm before.id e after=null.
    Esse é o caso que falha silenciosamente se o merge usar after.id diretamente.
    """
    conn = psycopg2.connect(postgres.get_connection_url())
    cur = conn.cursor()
    cur.execute("DELETE FROM orders WHERE id = 1")
    conn.commit()
    cur.close()
    conn.close()

    message = next(kafka_consumer)
    payload = message.value['payload']

    assert payload['op'] == 'd'
    assert payload['before']['id'] == 1
    assert payload['after'] is None
```

### 16.3 Testes de Regressão — Cenários Críticos

| Cenário | O que testar | Por que é crítico |
|---|---|---|
| DELETE event | `after` é `null`; merge usa `before.id` | Bug mais comum em merges CDC |
| Replay de eventos | Reprocessar os mesmos offsets produz o mesmo estado | Garantia de idempotência |
| Schema change (`ADD COLUMN`) | Novo campo aparece no evento; consumidores não quebram | Schema evolution |
| Conector reiniciado após pausa | Nenhum evento é perdido ou duplicado além do esperado | At-least-once delivery |
| Slot inativo (PostgreSQL) | Alertas disparam antes de `lag_bytes` atingir threshold crítico | Proteção contra disk full |
| Tombstone após DELETE | Consumidores com log compaction ignoram tombstones corretamente | Compactação de tópicos |
| Deduplicação MySQL | Offset composto `file+pos` deduplica no rollover de arquivo | Idempotência multi-banco |
| Deduplicação MongoDB | Resume token (`source.resume_token`) usado como chave única | Idempotência no MongoDB |

---

## 17. Desafios Práticos e Como Mitigá-los

### 17.1 Replication Slot Bloat — PostgreSQL

O maior risco operacional em CDC com PostgreSQL. Um slot inativo impede o descarte de segmentos WAL, causando crescimento ilimitado do disco.

- **Monitorar continuamente:** configure alertas quando `pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)` ultrapassar o threshold.
- **Safety net (PostgreSQL 13+):** defina `max_slot_wal_keep_size` para limitar o crescimento máximo.
- **Slot inativo prolongado:** drope o slot imediatamente (`SELECT pg_drop_replication_slot('dbz_slot')`) e recrie via snapshot incremental.

### 17.2 Schema Evolution

Adições de colunas precisam ser propagadas de forma compatível para downstream. Use Schema Registry com política `BACKWARD` ou `FULL`. Remoção e renomeação exigem coordenação coordenada — consulte a seção 13.

### 17.3 Consumer Lag

Configure alertas em Prometheus/Grafana para lag acima dos thresholds definidos pelo SLA. Métricas-chave: `kafka_consumer_lag` por grupo e tópico, `debezium_metrics_MilliSecondsBehindSource`, e crescimento de `lag_bytes` nos replication slots.

### 17.4 Tombstones e Log Compaction

Ao configurar tópicos CDC com `cleanup.policy=compact`, os tombstones (mensagens com valor `null`) publicados pelo Debezium após DELETE **não são removidos imediatamente**. Eles são retidos por `delete.retention.ms` (padrão: 24 horas) para que consumidores temporariamente offline ainda possam recebê-los antes da remoção definitiva.

Consumidores devem tratar tombstones explicitamente. Ignorar mensagens `null` sem processá-las pode levar a dados desatualizados: se o consumidor retomar após o `delete.retention.ms` expirar, a entrada já terá sido compactada sem que o DELETE tenha sido aplicado.

### 17.5 Tabelas sem Chave Primária

No PostgreSQL, use `REPLICA IDENTITY FULL` para capturar eventos de DELETE, mas esteja ciente do impacto no volume do WAL. No MySQL, o merge na Silver exige uma chave composta artificial.

```sql
-- PostgreSQL: captura completa em tabela sem PK (use com cautela)
ALTER TABLE legacy_table REPLICA IDENTITY FULL;
```

> ⚠ **Tabelas sem PK são um sinal de problema de modelagem.** Sempre que possível, adicione uma PK antes de configurar o CDC.

---

## 18. CDC em Arquiteturas de Data Lake

### 18.1 Comparativo dos Formatos

- **Delta Lake (Databricks/OSS):** `MERGE INTO` nativo com suporte a schema evolution e time travel. Maior adoção no ecossistema Spark. Compactação via `OPTIMIZE` e `VACUUM`. Artefato correto para Delta 3.x: `io.delta:delta-spark_2.12:3.x.x`.
- **Apache Iceberg:** suporte a múltiplos engines (Spark, Flink, Trino, Hive). MERGE via merge-on-read ou copy-on-write configurável por tabela. Excelente para cenários multi-engine.
- **Apache Hudi:** nativamente orientado a CDC com suporte a Merge On Read (MOR) e Copy On Write (COW). Recursos built-in de incremental queries e bootstrap a partir de datasets existentes.

### 18.2 Delta Lake — Controle de Concorrência e Otimização

O Delta Lake usa **controle de concorrência otimista** (Optimistic Concurrency Control — OCC). Múltiplas operações de MERGE podem ocorrer em paralelo, mas conflitos na mesma versão causam retentativa automática. Em pipelines CDC com alta frequência de micro-batches, isso pode gerar contenção:

```python
# Configurar número de retentativas em caso de conflito de concorrência
spark.conf.set("spark.databricks.delta.retryWriteConflicts.enabled", "true")
spark.conf.set("spark.databricks.delta.maxCommitAttempts", "10")
```

#### VACUUM — Retenção e Risco de Time Travel

O comando `VACUUM` remove arquivos Parquet antigos que não fazem mais parte da versão atual da tabela. O padrão de retenção é 7 dias. **Reduzir abaixo de 7 dias pode quebrar time travel e leituras concorrentes longas.**

```sql
-- Verificar retention atual
DESCRIBE DETAIL silver.orders;

-- VACUUM padrão (7 dias de retenção)
VACUUM silver.orders;

-- VACUUM com retenção customizada (mínimo recomendado: 7 dias)
VACUUM silver.orders RETAIN 168 HOURS;
```

> ⚠ **Em pipelines CDC com Spark Structured Streaming, o `VACUUM` não deve remover arquivos que ainda podem ser referenciados por checkpoints ativos.** Configure `delta.deletedFileRetentionDuration` com um valor maior que o intervalo máximo entre checkpoints do seu streaming job.

#### OPTIMIZE e ZORDER — Leitura Eficiente Pós-CDC

Pipelines CDC geram muitos pequenos arquivos Parquet (um por micro-batch). Sem compactação, as queries na Silver/Gold se tornam lentas por excesso de small files:

```sql
-- Compactar arquivos pequenos na Silver
OPTIMIZE silver.orders;

-- ZORDER colocaliza dados com o mesmo valor de coluna no mesmo arquivo,
-- acelerando filtros de leitura. Use nas colunas mais filtradas em queries downstream.
OPTIMIZE silver.orders ZORDER BY (customer_id, status);

-- Delta Lake 3.1+: Liquid Clustering substitui ZORDER com clustering adaptativo
-- Não exige OPTIMIZE manual; o clustering é feito incrementalmente.
ALTER TABLE silver.orders CLUSTER BY (customer_id, status);
```

> ℹ **Automatize `OPTIMIZE` em pipelines de produção.** Execute após cada N micro-batches ou em janela off-peak. Em Databricks, use `optimizeWrite=true` e `autoCompact=true` para reduzir small files automaticamente durante o MERGE.

### 18.2 Hudi — Design Orientado a CDC

- **Copy On Write (COW):** cada atualização reescreve o arquivo Parquet completo. Leituras rápidas, escritas mais custosas. Recomendado para cargas com mais leitura do que escrita.
- **Merge On Read (MOR):** atualizações são escritas em logs delta e mescladas na leitura. Ingestão de baixíssima latência, leituras ligeiramente mais custosas. Recomendado para alta frequência de CDC.

> ℹ **Hudi MOR — Compactação obrigatória em produção:** no modo MOR, o acúmulo de logs degrada a performance de leitura com o tempo. Use compactação **assíncrona** (via `HoodieCompactionConfig`) para mesclar logs em Parquet sem bloquear a ingestão. A compactação inline (`hoodie.compact.inline=true`) bloqueia cada commit até terminar — evite em produção com alta frequência de escrita.

### 18.3 Time Travel e Auditoria

```sql
-- Time travel em Delta Lake
SELECT * FROM silver.orders VERSION AS OF 10;
SELECT * FROM silver.orders TIMESTAMP AS OF '2024-03-15 10:00:00';

-- Time travel em Iceberg
SELECT * FROM silver.orders FOR SYSTEM_TIME AS OF TIMESTAMP '2024-03-15 10:00:00';

-- Time travel em Hudi
SELECT * FROM silver.orders
WHERE  _hoodie_commit_time <= '20240315100000';
```

---

## 19. CDC em Ambientes Cloud Gerenciados

| Serviço | Provider | Bancos Suportados | Destinos Nativos | Observação |
|---|---|---|---|---|
| AWS DMS | Amazon | Oracle, SQL Server, MySQL, PostgreSQL, MongoDB | S3, Redshift, Kinesis | Snapshot + CDC contínuo; latência de segundos a minutos — pode aumentar com Full LOB mode |
| Confluent Cloud | Confluent | PostgreSQL, MySQL, SQL Server, MongoDB, Oracle | Kafka nativo | Debezium gerenciado; latência < 1s |
| Azure Data Factory | Microsoft | SQL Server/Azure SQL, PostgreSQL, MySQL, Oracle | ADLS, Synapse, Azure SQL | Change Tracking para Azure SQL; log-based para demais |
| Google Datastream | Google | Oracle, MySQL, PostgreSQL | BigQuery, GCS, Spanner | Serverless; integração nativa com BigQuery; latência < 10s |
| Fivetran / Airbyte | SaaS / OSS | 50+ conectores | Warehouse, lakes | Airbyte open source; Fivetran gerenciado |

> ⚠ **AWS DMS e Full LOB mode:** necessário para colunas TEXT, BLOB e CLOB. Com esse modo habilitado, a latência pode subir de segundos para vários minutos. Avalie o tamanho de chunk e defina SLAs realistas.

---

## 20. Alta Disponibilidade e Multi-Região

Para pipelines CDC com requisito de alta disponibilidade:

- **MirrorMaker 2:** replicação cross-region de tópicos Kafka com sincronização de offsets. Habilite `offset.syncs` para que consumidores possam retomar do ponto correto após failover.
- **DR testado periodicamente:** simule failover de região em staging. A capacidade de recuperar de um DR não testado é zero.
- **Nunca dependa de cluster único** em produção crítica. Um cluster Kafka único é um ponto de falha para todos os pipelines CDC downstream.

```properties
# MirrorMaker 2 — configuração básica de replicação cross-region
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

- **Número de partições:** idealmente igual ao número de consumidores paralelos. Partições em excesso aumentam overhead.
- **Retenção no Kafka:** defina com base na janela de reprocessamento necessária. Para retenção longa, considere tiered storage.
- **Tamanho médio do evento:** some os campos before/after. Um evento típico tem de 200 bytes a vários KB.
- **Throughput esperado:** eventos/s × tamanho médio. Ex: 10.000 eventos/s × 1 KB = 10 MB/s ≈ 86 GB/dia.

### 21.2 Exemplo Numérico

Tabela de pedidos com 1 milhão de atualizações por dia, cada evento com 1 KB:
- Throughput médio: ~11,6 eventos/s ≈ 11,6 KB/s.
- Com retenção de 7 dias: 11,6 KB/s × 604.800 s ≈ 7 GB no Kafka.
- Com 10 tabelas similares: ~70 GB de armazenamento no Kafka.
- Custo estimado (AWS MSK, 3 brokers m5.large): ~$600/mês + EBS.

> ℹ **Comece com estimativa conservadora e monitore o crescimento real nos primeiros meses.** Ajuste retenção e partições conforme necessidade.

---

## 22. Monitoramento e Alertas

### 22.1 Métricas do Debezium (via JMX)

- `debezium_metrics_MilliSecondsBehindSource`: lag entre o evento no banco e publicação no Kafka.
- `debezium_metrics_TotalNumberOfEventsSeen`: total de eventos processados.
- `debezium_metrics_QueueRemainingCapacity`: capacidade restante da fila interna.

### 22.2 Métricas do Kafka

- **Consumer lag por grupo:** `kafka_consumer_lag` (Prometheus + kafka-exporter).
- **Taxa de produção/consumo:** `MessagesInPerSec`, `BytesInPerSec`.
- **Under-replicated partitions:** indica problemas de saúde do cluster.
- **Rebalance frequente:** pode indicar consumidores instáveis ou sessões expirando.

### 22.3 Métricas do Banco de Origem

- **PostgreSQL:** `lag_bytes` em `pg_replication_slots`. Alerte se > 50 GB ou tempo > 30 min.
- **MySQL:** espaço dos binlogs e arquivo mais antigo ainda necessário. Monitore `binlog_expire_logs_seconds`.
- **SQL Server:** tamanho da tabela de captura do CDC e latência de leitura.

### 22.4 Métricas do Spark

- **Batch duration por micro-batch:** se o tempo de processamento cresce consistentemente, o pipeline está perdendo para o volume de entrada.
- **State store size:** crescimento contínuo indica ausência de watermark ou watermark mal configurado. Em CDC, o state store cresce indefinidamente se `dropDuplicates` for usado sem `dropDuplicatesWithinWatermark`.
- **Checkpoint health e tempo de commit:** checkpoints lentos atrasam o próximo micro-batch e aumentam o lag.

> ⚠ **State store sem watermark — risco de OOM:** o state store do Spark Structured Streaming mantém em memória (e disco via RocksDB) todos os estados já vistos. Sem um watermark definido, o estado nunca é descartado — o que em pipelines CDC de longa duração resulta em crescimento ilimitado. O sintoma típico é Spark executors sendo mortos por OOM após dias ou semanas de execução. A solução é sempre usar `withWatermark` + `dropDuplicatesWithinWatermark` (Spark 3.5+) com uma janela que cobre o SLA máximo de reprocessamento:
> ```python
> # Exemplo: eventos atrasados em até 2 horas são aceitos; além disso, descartados
> deduped = (
>     parsed_df
>     .withWatermark('event_time', '2 hours')
>     .dropDuplicatesWithinWatermark(['_offset_key'])
> )
> ```
> Configure `event_time` a partir de `source.ts_ms` (timestamp do banco) — não do timestamp de ingestão no Kafka, que pode variar com reprocessamentos.

### 22.5 Alertas Recomendados

| Alerta | Condição | Ação |
|---|---|---|
| Replication slot lag | `lag_bytes > 50 GB` ou parado > 30 min | Investigar conector; dropar slot se necessário |
| Consumer lag | `lag > 10 min` (ou conforme SLA) | Escalar consumidores ou verificar gargalos |
| Conector parado | Status != RUNNING | Reiniciar via API / alertar equipe |
| Erros de DDL | Evento de DDL não processado | Verificar Schema Registry |
| Queda de throughput | Redução súbita > 50% | Verificar saúde do banco e conectores |
| Binlog/WAL próximo da expiração | > 80% da retenção consumida | Aumentar retenção ou investigar consumo lento |

---

## 23. Testes de Resiliência

Use Testcontainers ou ambientes de staging para simular:

1. **Banco de origem fica indisponível:** parar o container do banco e reiniciar. Verificar se o conector retoma do offset correto.
2. **Kafka fora do ar:** simular falha nos brokers. O conector deve reconectar com backoff exponencial.
3. **Reinicialização do conector:** matar o processo do Kafka Connect e verificar se offsets são restaurados sem perda.
4. **Corrupção de offset:** forçar offset inválido e verificar se o conector se recupera via snapshot.
5. **Sobrecarga de mensagens:** gerar picos de inserções e observar se o lag aumenta de forma controlada.
6. **Deduplicação:** enviar mensagens duplicadas (mesmo offset) e confirmar que a Silver não as aplica duas vezes.

**Ferramentas:**
- **Testcontainers** para orquestrar banco, Kafka e Debezium em testes automatizados.
- **Apache JMeter** ou **k6** para gerar carga no banco.
- **Chaos Mesh** ou **Gremlin** para injeção de falhas em Kubernetes.

> ℹ **Automatize os testes de resiliência em CI.** A resiliência não é um evento único — é uma qualidade que deve ser continuamente validada. Um pipeline CDC sem testes automatizados é um pipeline que vai quebrar silenciosamente em produção.

---

## 24. Matriz de Garantias

| Camada | Garantia | Responsabilidade |
|---|---|---|
| Banco | Ordem transacional | WAL / Binlog / CDC nativo |
| Debezium | Captura consistente, at-least-once | Offset interno + replication slot |
| Kafka | **At-least-once por padrão**; exactly-once se producer transacional + `isolation.level=read_committed` + Connect configurado | Broker + retenção configurada |
| Spark | Processamento incremental com checkpointing | Checkpoint + foreachBatch |
| Delta Lake / Iceberg / Hudi | Idempotência via MERGE | Schema + chave primária + _offset_key |

---

## 25. Incidentes Reais Comuns

| Problema | Causa Raiz | Mitigação |
|---|---|---|
| Disk full no banco | Replication slot não consumido (PostgreSQL) / Binlog acumulado | Alertas de lag + `max_slot_wal_keep_size` |
| Snapshot inesperado | Binlog/WAL expirado enquanto conector estava parado | Aumentar retenção; monitorar inatividade |
| Duplicidade silenciosa | Rebalance de consumidor + sink não idempotente | MERGE idempotente + deduplicação por offset |
| Quebra de consumidores downstream | `DROP COLUMN` ou `RENAME COLUMN` sem coordenação | Schema Registry + ciclo de vida explícito |
| Ordering quebrado | PK alterada ou message key customizada incorretamente | Nunca alterar PK sem plano; validar transforms |
| DELETE não aplicado na Silver | Merge usando `after.id` sem `CASE WHEN` | Correção com `CASE WHEN b.op = 'd'` |
| OOM no Spark | `dropDuplicates` sem watermark em stream contínuo | `dropDuplicatesWithinWatermark` (Spark 3.5+) |
| Conector não inicia após migração 1.x → 2.x | `database.dbname` → `database.names` não atualizado | Validar todas as propriedades após upgrade |
| Offset inválido ao reiniciar Spark após Kafka expirar retenção | Checkpoint Spark aponta para offset que não existe mais no tópico | Aumentar retenção Kafka ou ajustar `startingOffsets` para retomar do mais antigo disponível |
| Ordering inconsistente em pipeline multi-região | Clock skew entre data centers afeta ordenação por `source.ts_ms` | Usar LSN/offset como critério de ordenação primário, não timestamp; NTP sincronizado |
| Executor Spark morto por OOM em transação gigante | Uma única transação gera batch enorme que não cabe em memória | Limitar `max_batch_size` no conector; monitorar tamanho de batches; usar `fetch.min.bytes` |
| Silver corrompida após mudança de PK | Eventos com PK antiga e nova são tratados como entidades distintas | Alterar PK = reprocessamento full; planejar como migração, não como DDL normal |

### 25.1 Riscos Específicos em Produção Enterprise

#### Clock Skew entre Regiões

Em pipelines multi-região com MirrorMaker 2, o campo `source.ts_ms` representa o timestamp do commit no banco de origem. Se existir clock skew entre servidores (mesmo que pequeno — dezenas de milissegundos), eventos de regiões diferentes com timestamps similares podem chegar fora de ordem no consumidor consolidado. **Nunca use `source.ts_ms` como critério único de ordenação** em pipelines cross-region. Use o offset Kafka (por partição) ou o LSN (por banco) como critério primário.

#### Reprocessamento após Retenção Kafka Expirar

Se o Spark Structured Streaming ficar parado enquanto a retenção do tópico Kafka expira, ao retomar o job os checkpoints apontarão para offsets que não existem mais no broker. O Spark lançará `OffsetOutOfRangeException`. A solução é:
```python
# Recuperação: reiniciar do offset mais antigo disponível
spark.readStream.format("kafka") \
    .option("subscribe", "orders") \
    .option("startingOffsets", "earliest") \
    .option("failOnDataLoss", "false")  # permite continuar mesmo com gap
```
> ⚠ `failOnDataLoss=false` significa que eventos perdidos não causam falha — mas também não são reprocessados. Avalie se a perda é aceitável; se não for, a Bronze imutável é o ponto de replay.

#### Large Transactions — Risco de Pressão de Memória

Transações que modificam milhares ou milhões de linhas em um único commit geram um batch Kafka igualmente grande. Isso pode:
- Pressionar a memória do consumer (Spark executors)
- Bloquear o WAL do PostgreSQL até o batch ser confirmado
- Causar latência percebida de minutos em outros consumidores do mesmo tópico

Mitigações: limitar `max.poll.records` no consumer, monitorar `pg_stat_activity` para transações longas no banco de origem, e considerar quebrar operações bulk em batches menores na aplicação produtora.

---

## 26. Troubleshooting Comum

### 26.1 Conector não inicia

- Verifique configurações de banco (host, porta, credenciais).
- Confirme que o banco está com log habilitado (`wal_level=logical`, `binlog_format=ROW`).
- No PostgreSQL, verifique se o replication slot já existe e está ativo. Se necessário, drope manualmente:
  ```sql
  SELECT pg_drop_replication_slot('dbz_slot');
  ```
- No Debezium 2.x com MySQL/SQL Server, confirme que `database.names` (plural) está sendo usado, não `database.dbname`.
- No Kafka Connect, verifique se o worker pode acessar os tópicos de offset e histórico.

### 26.2 Eventos não chegam ao Kafka

- Verifique o status do conector: `GET /connectors/{name}/status`.
- Monitore o lag no banco; lag alto pode indicar problemas de rede ou desempenho.
- Confirme que o tópico de destino foi criado e tem partições suficientes.

### 26.3 Eventos de DELETE não são aplicados na Silver

- O merge provavelmente está usando `after.id` diretamente. Corrija com `CASE WHEN b.op = 'd' THEN b.before.id ELSE b.after.id END` e verifique que `whenMatchedDelete` está declarado antes de `whenMatchedUpdate`.

### 26.4 Duplicatas na Silver

- Para MySQL, confirme que o offset composto `source.file + ':' + source.pos` está sendo usado (não apenas `source.pos`).
- Confirme que `_offset_key` está sendo armazenado e usado como critério de deduplicação no MERGE.

### 26.5 Schema incompatível no Schema Registry

- Verifique a política de compatibilidade. Colunas adicionadas sem `DEFAULT` podem quebrar consumidores com política `BACKWARD`.
- Sempre adicione colunas com valor `DEFAULT` no banco antes de evoluir o schema.

### 26.6 Disk Full no PostgreSQL devido ao WAL

- O replication slot está parado. Drope o slot ou aumente `max_slot_wal_keep_size`.
- Configure alertas para `lag_bytes` antes de atingir o threshold crítico.

### 26.7 OOM no Spark após execução prolongada

- `dropDuplicates` sem watermark acumula estado indefinidamente. Migre para `dropDuplicatesWithinWatermark` (Spark 3.5+) com uma janela adequada ao SLA de reprocessamento.

---

## 27. Conclusão

CDC log-based combinado com Kafka e arquitetura em camadas (Bronze → Silver → Gold) é o padrão de referência para pipelines de dados modernos que exigem baixa latência, confiabilidade e auditoria completa.

CDC em produção não é sobre capturar eventos. É sobre garantia de ordenação, recuperação determinística, idempotência real, monitoramento ativo e governança de schema.

Os pilares de um pipeline CDC maduro:

- **Imutabilidade da Bronze:** fonte única de verdade e ponto de replay para qualquer reprocessamento futuro.
- **Debezium com particionamento correto:** chave primária como message key garante ordering por entidade e paralelismo seguro.
- **Merge idempotente na Silver:** `whenMatchedDelete` antes de `whenMatchedUpdate`, `CASE WHEN` para identificador de entidade, deduplicação por `_offset_key` adequado ao banco.
- **Snapshot incremental:** eliminação do lock global via chunk-based snapshot com watermark interno.
- **Schema evolution gerenciada:** Schema Registry com política `BACKWARD` ou `FULL`, e ciclo de vida explícito para operações destrutivas de DDL.
- **Segurança e PII desde a origem:** mascaramento no conector, crypto shredding antes da Bronze, TLS em trânsito, criptografia em repouso.
- **Monitoramento proativo:** replication slot lag, consumer lag, espaço de Binlog e alertas de schema incompatível.
- **Testes automatizados e de resiliência:** suíte cobrindo DELETEs, replay, schema change e falhas de infraestrutura.
- **Alta disponibilidade:** MirrorMaker 2 para multi-região, DR testado periodicamente.
- **Versões atualizadas:** Delta Lake 3.x (`delta-spark`), Kafka 3.7+, Debezium 2.5+, Spark 3.5+.

Quando bem implementado: Data Lake incremental consistente, baixa latência, reprocessamento seguro e arquitetura escalável. Quando mal implementado: corrupção silenciosa, perda de confiança no dado e incidentes recorrentes. A diferença está no rigor arquitetural.

---

## Glossário

| Termo | Significado |
|-------|-------------|
| **WAL** | Write-Ahead Log (PostgreSQL) |
| **Binlog** | Binary Log (MySQL) |
| **LSN** | Log Sequence Number (PostgreSQL, SQL Server) |
| **Resume Token** | Identificador de checkpoint no MongoDB Change Streams; único mesmo após failover |
| **Replication Slot** | Mecanismo do PostgreSQL que retém WAL até ser consumido pelo conector |
| **Offset** | Posição no log que permite retomada exata após falha |
| **Schema Registry** | Serviço que armazena versões de schemas Avro/JSON/Protobuf com política de compatibilidade |
| **PII** | Personally Identifiable Information |
| **CDC** | Change Data Capture |
| **Debezium** | Conector open source de referência para CDC log-based |
| **Kafka Connect** | Framework para conectar Kafka a sistemas externos de forma distribuída |
| **Delta Lake / Iceberg / Hudi** | Formatos de tabela open source para data lakes com suporte a upsert/delete |
| **GTID** | Global Transaction Identifier (MySQL); identifica transações globalmente na topologia |
| **Tombstone** | Mensagem com valor null publicada após DELETE para sinalizar compactação |
| **MirrorMaker 2** | Ferramenta Kafka para replicação cross-cluster/cross-region com sincronização de offsets |
| **dropDuplicatesWithinWatermark** | Operação Spark 3.5+ para deduplicação com janela temporal limitada, evitando estado ilimitado |
| **Crypto shredding** | Técnica de direito ao esquecimento: destruir a chave de criptografia torna dados históricos irrecuperáveis |
| **`delete.retention.ms`** | Parâmetro Kafka que controla por quanto tempo tombstones são retidos em tópicos compactados antes da remoção definitiva (padrão: 86400000 ms = 24h) |
| **`max_logical_replication_workers`** | Parâmetro PostgreSQL que limita o número de workers para replicação lógica; deve ser dimensionado junto com `max_worker_processes` para evitar falha silenciosa de conectores |

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
