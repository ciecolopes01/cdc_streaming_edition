# Change Data Capture — Streaming Edition

---

## Sumário

1. [O Que É CDC de Verdade](#1-o-que-é-cdc-de-verdade)
2. [Quando Não Usar CDC Log-based](#2-quando-não-usar-cdc-log-based)
3. [Níveis de Maturidade do CDC](#3-níveis-de-maturidade-do-cdc)
4. [Arquitetura CDC com Kafka e Streaming](#4-arquitetura-cdc-com-kafka-e-streaming)
5. [Camada Bronze — A Fundação Imutável](#5-camada-bronze--a-fundação-imutável)
6. [Exemplos Reais por Banco de Dados](#6-exemplos-reais-por-banco-de-dados)
7. [Formato de Evento no Tópico Kafka](#7-formato-de-evento-no-tópico-kafka)
8. [Camadas Silver e Gold](#8-camadas-silver-e-gold)
9. [Semânticas de Entrega e Idempotência](#9-semânticas-de-entrega-e-idempotência)
10. [Snapshot Inicial — Consistência e Estratégias](#10-snapshot-inicial--consistência-e-estratégias)
11. [Schema Evolution — DDL no Banco de Origem](#11-schema-evolution--ddl-no-banco-de-origem)
12. [Segurança, PII e Conformidade](#12-segurança-pii-e-conformidade)
13. [Modos de Deploy do Debezium](#13-modos-de-deploy-do-debezium)
14. [Testando Pipelines CDC](#14-testando-pipelines-cdc)
15. [Desafios Práticos e Como Mitigá-los](#15-desafios-práticos-e-como-mitigá-los)
16. [CDC em Arquiteturas de Data Lake](#16-cdc-em-arquiteturas-de-data-lake)
17. [CDC em Ambientes Cloud Gerenciados](#17-cdc-em-ambientes-cloud-gerenciados)
18. [Dimensionamento e Custos](#18-dimensionamento-e-custos)
19. [Monitoramento e Alertas](#19-monitoramento-e-alertas)
20. [Testes de Resiliência](#20-testes-de-resiliência)
21. [Troubleshooting Comum](#21-troubleshooting-comum)
22. [Conclusão](#22-conclusão)

---

## 1. O Que É CDC de Verdade

Change Data Capture (CDC) é a prática de capturar eventos de modificação de dados — INSERT, UPDATE e DELETE — diretamente dos mecanismos internos do banco de dados, preservando a ordem de commit, a consistência transacional e a capacidade de reprocessamento.

Ao contrário de abordagens baseadas em polling (consultas periódicas), o CDC orientado a log não depende de colunas de controle nem de timestamps. Ele lê o transaction log do banco — o mesmo registro que garante durabilidade e recuperação de falhas — transformando cada operação em um evento rastreável e reproduzível.

> ℹ **CDC maduro = log-based + checkpoint por offset + capacidade de replay desde qualquer ponto histórico.**

Os três pilares de um pipeline CDC robusto são:

- **Captura fiel:** cada linha modificada gera exatamente um evento com os estados before e after.
- **Offset rastreado:** a posição de leitura no log é persistida, garantindo retomada sem perda além do esperado.
- **Replay confiável:** é possível reprocessar eventos históricos para reconstruir estados, alimentar novos consumidores ou corrigir erros de transformação.

---

## 2. Quando Não Usar CDC Log-based

CDC log-based é a solução certa para a maioria dos pipelines de dados em tempo real — mas não para todos os cenários. Aplicar a receita no lugar errado gera complexidade sem benefício correspondente.

**Evite CDC log-based quando:**

- **A tabela não tem chave primária definida.** O Debezium depende da chave primária para compor a message key do Kafka e garantir ordering por entidade. Tabelas sem PK podem ser capturadas em modo `replica identity FULL` no PostgreSQL, mas isso aumenta drasticamente o volume do WAL e pode degradar o banco. Em MySQL, eventos de UPDATE e DELETE em tabelas sem PK são publicados sem identificador único, tornando o merge na Silver extremamente difícil.

- **A fonte de dados não expõe o transaction log.** Serviços SaaS (Salesforce, HubSpot, Stripe, Shopify) não oferecem acesso ao log interno. Nesses casos, use conectores baseados em API incremental (Airbyte, Fivetran) ou webhooks nativos do serviço.

- **O volume de dados é baixo e a latência tolerada é alta.** Para tabelas pequenas com poucas escritas e SLA de horas, o custo operacional de manter Debezium + Kafka + replication slot não se justifica. Um simples job de polling com `updated_at` atende com muito menos complexidade.

- **O banco de dados não suporta log-based CDC.** SQLite, Microsoft Access e alguns bancos embarcados não possuem mecanismo de transaction log exposto. Bancos em planos gerenciados restritivos (ex: algumas instâncias RDS com parâmetros fixos) podem não permitir as configurações necessárias (`wal_level=logical`, `binlog_format=ROW`).

- **O schema muda com altíssima frequência de forma destrutiva.** Pipelines com `DROP COLUMN` e `RENAME COLUMN` recorrentes exigem coordenação contínua entre produtor e consumidores. Se o schema é altamente volátil por design, considere event sourcing na camada de aplicação como alternativa.

> ℹ **O critério decisivo:** se você precisa de baixa latência, captura de DELETEs, rastreabilidade de toda a história de uma entidade e o banco suporta log-based CDC — use CDC log-based. Nos demais casos, avalie a alternativa mais simples primeiro.

---

## 3. Níveis de Maturidade do CDC

A escolha da estratégia de CDC determina diretamente o nível de confiabilidade, a sobrecarga no banco de dados e os tipos de eventos capturáveis. A tabela abaixo resume os quatro níveis principais:

| Nível | Estratégia | Captura DELETE? | Ordem Transacional? | Impacto no Banco |
|---|---|---|---|---|
| 1 — Watermark | Coluna `updated_at` | ❌ Não | ❌ Não | Baixo |
| 2 — Change Tracking¹ | Metadados SQL Server | ✅ Sim | Parcial | Baixo |
| 3 — Trigger-based | Triggers de auditoria | ✅ Sim | Parcial | Alto |
| 4 — Log-based | WAL / Binlog / CDC nativo | ✅ Sim | ✅ Sim | Mínimo |

> ¹ Change Tracking é um recurso exclusivo do Microsoft SQL Server que registra *quais* linhas mudaram, mas não captura os valores anteriores. Não deve ser confundido com o CDC nativo (nível 4).

### 3.1 Nível 1 — Watermark

A estratégia mais simples: consultas periódicas filtradas por uma coluna `updated_at`. Não captura deleções físicas e não garante ordem transacional. Adequada apenas para cenários de baixa criticidade.

### 3.2 Nível 2 — Change Tracking (SQL Server) vs CDC Nativo

Ponto de atenção importante: o Microsoft SQL Server oferece dois recursos com nomes semelhantes, mas mecanismos distintos. Compreender a diferença é fundamental para evitar configurações incorretas:

- **Change Tracking (Nível 2):** registra apenas *quais* linhas foram alteradas e o tipo de operação, sem armazenar os valores anteriores das colunas. Não entrega o estado `before` — essencial para pipelines CDC completos.
- **CDC nativo (Nível 4, usado pelo Debezium):** lê diretamente o transaction log e captura o estado completo `before` e `after` de cada operação, incluindo todos os valores de coluna.

> ℹ **O Debezium para SQL Server utiliza o CDC nativo (`sys.sp_cdc_enable_table`), e não o Change Tracking. Certifique-se de habilitar o recurso correto em produção.**

### 3.3 Nível 3 — Trigger-based

Triggers gravam cada operação em tabelas de auditoria. Funciona em qualquer banco relacional, porém aumenta significativamente a latência das transações originais e cria acoplamento entre o pipeline de dados e o schema da aplicação. Escala mal em cargas de escrita elevadas.

### 3.4 Nível 4 — Log-based (Padrão Enterprise)

Leitura direta do transaction log: WAL no PostgreSQL, Binlog no MySQL, CDC nativo no SQL Server. Modelo recomendado para produção: impacto mínimo nas transações originais, captura completa e suporte nativo a replay por offset.

> ℹ **Debezium** é o conector open source de referência da indústria para CDC log-based. Suporta PostgreSQL (WAL), MySQL (Binlog), SQL Server (CDC nativo), MongoDB (Change Streams), Oracle e outros, publicando eventos diretamente em tópicos Kafka.

---

## 4. Arquitetura CDC com Kafka e Streaming

```
[ Banco de Dados: PostgreSQL / MySQL / SQL Server / MongoDB ]
          |
          v
[ Debezium Connector ]  ←  lê o transaction log continuamente
          |
          v
[ Kafka Topic por Tabela ]  ex: dbserver1.dbo.orders
          |
    +------+-------+-------+
    |              |       |
    v              v       v
[ Bronze Lake ] [Flink]  [ML Pipeline]
    |
    v
[ Silver: merge/dedup ]
    |
    v
[ Gold: modelo analítico ]
```

O conector CDC lê o log do banco e publica cada mudança como um evento imutável em tópicos Kafka. Cada tabela corresponde a um tópico separado. Consumidores podem ser batch, streaming, pipelines de ML ou qualquer sistema downstream — todos completamente desacoplados entre si e do banco de origem.

### 4.1 Kafka como Backbone — Benefícios e Configurações-Chave

- **Retenção configurável:** eventos ficam disponíveis por horas, dias ou indefinidamente, permitindo que novos consumidores façam backfill completo.
- **Particionamento por chave primária:** o Debezium usa a chave primária da tabela como *message key* do Kafka. Isso garante que todas as mudanças de uma mesma linha caiam sempre na mesma partição, preservando a ordem de eventos por entidade. Consumidores que precisam aplicar mudanças em ordem correta dependem fundamentalmente dessa garantia.
- **Schema Registry:** integração com Confluent Schema Registry ou AWS Glue Schema Registry para versionamento de schemas e compatibilidade evolutiva entre produtor e consumidores.
- **Exactly-once com transações Kafka:** configurando `enable.idempotence=true` no produtor e processamento stateful com commit atômico (ex: Flink com checkpointing), é possível obter exactly-once end-to-end.

> ℹ **Por padrão, o Debezium configura a chave da mensagem Kafka com a chave primária da tabela.** Isso pode ser customizado via `transforms` no conector, mas alterar sem cuidado quebra a garantia de ordering por entidade.

---

## 5. Camada Bronze — A Fundação Imutável

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

## 6. Exemplos Reais por Banco de Dados

### 6.1 SQL Server — CDC Nativo + Debezium

```sql
-- 1. Habilitar CDC no banco de dados
EXEC sys.sp_cdc_enable_db;

-- 2. Habilitar CDC na tabela alvo (CDC nativo, não Change Tracking)
EXEC sys.sp_cdc_enable_table
    @source_schema        = 'dbo',
    @source_name          = 'orders',
    @role_name            = NULL,
    @supports_net_changes = 0;  -- Debezium usa funções "all changes"; net changes não é necessário

-- 3. Verificar que a tabela está com CDC habilitado
SELECT name, is_cdc_enabled
FROM   sys.tables
WHERE  name = 'orders';
```

> ℹ O parâmetro `@supports_net_changes = 1` habilita funções de "net changes" que retornam apenas o estado final após múltiplas alterações. O Debezium utiliza as funções de "all changes" (`cdc.fn_cdc_get_all_changes_...`), portanto `supports_net_changes` pode ficar como 0 (padrão) sem impacto no funcionamento.
>
> ⚠ **Atenção para ambientes com uso misto:** se outras aplicações além do Debezium consomem o CDC nativo do SQL Server via `fn_cdc_get_net_changes_*`, habilite `@supports_net_changes = 1` para não quebrar esses consumidores. O Debezium continuará funcionando normalmente, pois não depende das funções de net changes.

Cada mudança gera um registro com LSN (Log Sequence Number) único. O Debezium persiste o LSN processado como offset, garantindo retomada exata após falhas.

> ⚠ **Configure a retenção do CDC no SQL Server** (`cdc.cleanup_change_table`) de acordo com a latência máxima tolerada do conector. O padrão é 3 dias — insuficiente para cenários com janelas de manutenção longas ou conectores pausados.

### 6.2 PostgreSQL — Logical Decoding + WAL

```sql
-- 1. Configurar WAL level (requer reinicialização do servidor)
ALTER SYSTEM SET wal_level            = logical;
ALTER SYSTEM SET max_replication_slots = 10;
ALTER SYSTEM SET max_wal_senders       = 10;

-- 2. Criar replication slot dedicado ao Debezium
SELECT pg_create_logical_replication_slot(
    'debezium_orders',
    'pgoutput'     -- nativo; ou 'wal2json' se instalado
);

-- 3. Monitorar o lag do slot (executar periodicamente)
SELECT slot_name,
       active,
       pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes
FROM   pg_replication_slots;
```

> ✗ **Risco crítico:** um replication slot não consumido acumula WAL indefinidamente, podendo causar *disk full* e queda total do banco. Configure alertas quando `lag_bytes` ultrapassar o threshold definido. Se o conector parar por longo período, avalie dropar o slot e recriar via snapshot inicial do Debezium.

> ⚠ **PostgreSQL 13+:** configure `max_slot_wal_keep_size` como safety net para limitar o crescimento máximo de WAL retido por slots, evitando *disk full* em casos de conector inativo.

### 6.3 MySQL / MariaDB — Binlog ROW Format

```ini
# ── MySQL 5.7 ──────────────────────────────────────────────────────────
# my.cnf / my.ini
[mysqld]
server-id        = 1           # Único por instância na topologia
log_bin          = mysql-bin   # Habilita o Binlog
binlog_format    = ROW         # Obrigatório para CDC
binlog_row_image = FULL        # Garante before/after completos
expire_logs_days = 7           # Retenção (deprecated no MySQL 8.0)

# ── MySQL 8.0+ ──────────────────────────────────────────────────────────
# expire_logs_days está deprecated desde 8.0 e REMOVIDO no MySQL 8.4 em diante
binlog_expire_logs_seconds = 604800  # 7 dias em segundos
```

O offset é composto pelo nome do arquivo de Binlog mais a posição dentro dele (ex: `mysql-bin.000123:4567`). O Debezium persiste esse par como checkpoint após cada batch confirmado.

> ⚠ **No MySQL 8.0+, use `binlog_expire_logs_seconds` no lugar de `expire_logs_days` (deprecated). A partir do MySQL 8.4, `expire_logs_days` foi completamente removido** — configurá-lo causará erro de inicialização do servidor; utilize exclusivamente `binlog_expire_logs_seconds`. Em qualquer versão, o valor deve ser maior que o tempo máximo de inatividade tolerado para o conector: se ele ficar parado além desse período, os logs serão descartados e um novo snapshot completo será necessário.

### 6.4 MongoDB — Change Streams

```javascript
// Escutar mudanças na coleção orders
const changeStream = db.orders.watch([
    { $match: { operationType: { $in: ['insert','update','delete'] } } }
], {
    fullDocument: 'updateLookup',  // inclui documento completo após update
    resumeAfter: savedResumeToken  // retoma do checkpoint salvo
});

changeStream.on('change', (event) => {
    // IMPORTANTE: salvar o resume token como checkpoint
    const checkpoint = event._id;
    processEvent(event.operationType, event.fullDocument, checkpoint);
});
```

> ℹ **O MongoDB exige replica set ativo** para habilitar Change Streams, mesmo em ambientes de desenvolvimento com instância única. Use `rs.initiate()` antes de configurar o conector Debezium.

---

## 7. Formato de Evento no Tópico Kafka

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

**Valores de `op`:** `c` = insert · `u` = update · `d` = delete · `r` = snapshot

**Regra de ouro:** `after` é `null` para DELETE e `before` é `null` para INSERT. Qualquer código que acesse `b.after.id` sem verificar o tipo de operação falhará silenciosamente em eventos de DELETE.

> ✗ **Atenção crítica:** nunca acesse diretamente `b.after.id` em merges — use `CASE WHEN b.op = 'd' THEN b.before.id ELSE b.after.id END`. Detalhado na seção 8.1.

---

## 8. Camadas Silver e Gold

### 8.1 Silver — Merge Incremental Correto e Idempotente

A Silver aplica a semântica transacional sobre os eventos crus da Bronze, produzindo uma visão consistente e atualizada de cada entidade. O conceito de at-least-once delivery — e o impacto direto na deduplicação — é tratado em detalhes na seção 9.1.

**✗ Bug comum — acesso direto a `after.id`:**

```python
# ❌ INCORRETO — falha silenciosamente em eventos de DELETE
# Em DELETEs, after = null, portanto b.after.id = null
# A condição nunca fará match e o DELETE nunca será aplicado na Silver
.merge(
    batch_df.alias('b'),
    condition = 's.id = b.after.id'   # ERRADO: after é null em DELETEs
)
```

**Solução correta — CASE WHEN por tipo de operação, com deduplicação robusta por banco:**

```python
from delta.tables import DeltaTable
from pyspark.sql import functions as F

def build_offset_key(df, source_type: str):
    """
    Constrói chave de deduplicação conforme o banco de origem.
    PostgreSQL / SQL Server : source.lsn  (inteiro único)
    MySQL / MariaDB         : source.file + ':' + source.pos  (composto)
    MongoDB                 : source.resume_token  (string completa do resume token)
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

    # 2. Deduplicação: mantém apenas o último evento por chave de offset.
    #    Em at-least-once, o mesmo offset pode chegar mais de uma vez após falha.
    #
    #    ATENÇÃO — dropDuplicates em Structured Streaming acumula estado indefinidamente.
    #    Para streams de longa duração, prefira dropDuplicatesWithinWatermark (Spark 3.5+)
    #    com um watermark adequado ao SLA de reprocessamento:
    #
    #    deduped = (
    #        deduped
    #        .withWatermark('event_time', '2 hours')
    #        .dropDuplicatesWithinWatermark(['_offset_key'])
    #    )
    #
    #    Usar dropDuplicates sem watermark em streams contínuos pode causar OOM
    #    à medida que o estado cresce sem limite. Em foreachBatch (modo micro-batch),
    #    o risco é menor pois o estado é por batch — mas monitore o tamanho do checkpoint.
    deduped = deduped.dropDuplicates(['_offset_key'])

    # 3. MERGE com ordem correta das cláusulas:
    #    whenMatchedDelete ANTES de whenMatchedUpdate evita ambiguidade
    #    quando a linha existe e o evento é DELETE.
    #    Se a cláusula de UPDATE viesse primeiro, um evento DELETE poderia acidentalmente
    #    ser interpretado como UPDATE se a condição não filtrasse corretamente.
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
        condition="b.op = 'd'"                    # ← DELETE primeiro
    ).whenMatchedUpdate(
        condition="b.op IN ('u', 'c')",           # ← UPDATE depois
        set={
            'status':      'b.after.status',
            'total':       'b.after.total',
            'updated_at':  'b.source.ts_ms',
            '_offset_key': 'b._offset_key'        # armazena para auditoria
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
    .start()
```

> ℹ **Por que `whenMatchedDelete` antes de `whenMatchedUpdate`?** O Delta Lake processa as cláusulas `whenMatched` na ordem em que são declaradas. Colocar o DELETE primeiro garante que um evento `op='d'` seja aplicado como deleção, sem risco de cair acidentalmente na cláusula de UPDATE caso a lógica de condição tenha uma edge case. É uma questão de clareza e previsibilidade, não apenas de estilo.

> ℹ **Armazenar `_offset_key` na Silver** cumpre dois objetivos: (1) rastreabilidade — é possível saber exatamente qual evento originou o estado atual de cada linha; (2) idempotência — em cenários de at-least-once, o `dropDuplicates` por offset garante que o mesmo evento não seja aplicado duas vezes, mesmo após falhas e retomadas.

> ⚠ **A chave de offset no MySQL é composta (`source.file + ':' + source.pos`)**, não apenas o `pos`. Usar somente `source.pos` como chave de deduplicação pode gerar falsos positivos quando um novo arquivo de Binlog é criado (o `pos` reinicia do zero para o novo arquivo). Sempre concatene os dois campos.

> ⚠ **Para MongoDB, o campo correto é `source.resume_token`.** O campo `source.ord` é um número sequencial, mas o resume token completo é a string em `source.resume_token`, que garante a unicidade mesmo após failover.

### 8.2 Gold — Modelo Analítico Pronto para Consumo

A Gold entrega agregações, dimensões e fatos modelados para consumo por ferramentas de BI, APIs ou modelos de ML. Pode ser reconstruída a qualquer momento a partir da Bronze, graças à imutabilidade do histórico.

- **Modelos dimensionais:** Star Schema ou Snowflake Schema para ferramentas de BI como Power BI, Tableau e Looker.
- **Feature stores:** tabelas atualizadas em near real-time para alimentar modelos de ML, usando eventos da Silver como fonte.
- **APIs de leitura:** dados servidos com latência de segundos, construídos sobre a Gold com SLA definido.

---

## 9. Semânticas de Entrega e Idempotência

### 9.1 At-Least-Once — Padrão Debezium

O Debezium garante *at-least-once delivery*: em caso de falha e retomada, alguns eventos podem ser republicados. Consumidores devem ser idempotentes — processar o mesmo evento duas vezes deve produzir o mesmo resultado que processar uma vez.

Na prática, idempotência na Silver é implementada via:

- **Deduplicação por offset:** use a `_offset_key` adequada para o banco conforme demonstrado na seção 8.1.
- **MERGE com chave primária:** a operação de MERGE é naturalmente idempotente para UPDATEs e INSERTs — aplicar o mesmo evento duas vezes produz o mesmo estado final.
- **Tombstone para DELETEs:** o Debezium publica dois eventos para um DELETE: o evento com `op='d'` e um tombstone (valor `null`) que permite que o compactador do Kafka remova a entrada do log compactado.

### 9.2 Exactly-Once com Kafka Transactions

Para cenários que exigem exactly-once end-to-end, é necessário combinar:

- Produtores transacionais no Kafka (`enable.idempotence=true`)
- Processamento stateful com commit atômico de offset e escrita

> ℹ **Apache Flink com Kafka source e Delta Lake sink** oferece exactly-once nativo via two-phase commit. Para isso, adicione a dependência `io.delta:delta-flink` ao projeto (atenção: o artifact correto a partir do Delta Lake 3.x é `delta-flink`, não `delta-connectors-flink` — o nome foi atualizado quando o projeto foi migrado para o repositório principal do Delta), configure o sink com `.option("checkpointLocation", "...")` e habilite o checkpointing do Flink no modo `EXACTLY_ONCE`. Para pipelines baseados em Kafka Streams, utilize `processing.guarantee=exactly_once_v2` (disponível a partir do Kafka 2.5), que combina transações Kafka com processamento idempotente para garantir exactly-once end-to-end sem necessidade de infraestrutura adicional. Em versões anteriores ao Kafka 2.5, o valor equivalente é `exactly_once`, que oferece a mesma garantia semântica porém com menor throughput por não aproveitar as otimizações de desempenho introduzidas no `exactly_once_v2`.

---

## 10. Snapshot Inicial — Consistência e Estratégias

### 10.1 O Problema de Consistência

Ao conectar o Debezium a uma tabela existente com dados históricos, ele precisa realizar um snapshot inicial — leitura completa da tabela — antes de começar a capturar eventos do log. O desafio central é garantir que não haja lacuna nem duplicação entre o snapshot e os eventos de log que ocorrem durante a sua execução.

### 10.2 Snapshot Tradicional (com Lock)

No modo padrão, o Debezium adquire um lock de leitura na tabela durante o snapshot para garantir que o LSN de início seja coerente com os dados lidos. Após o snapshot, o conector retoma a partir desse LSN, garantindo zero gap.

> ⚠ **O lock de snapshot pode causar bloqueio temporário** em tabelas de alta concorrência. Em ambientes de produção críticos, prefira o snapshot incremental.

### 10.3 Snapshot Incremental (Chunk-based — Recomendado)

Disponível a partir do Debezium 1.6, o snapshot incremental elimina a necessidade de lock global. A tabela é lida em chunks por faixa de chave primária, e o mecanismo de watermark garante que eventos de log recebidos durante o snapshot sejam mesclados corretamente.

```json
{
  "snapshot.mode": "initial",
  "incremental.snapshot.chunk.size": "1024",
  "signal.data.collection": "debezium.signals"
}
```

```sql
-- Disparar snapshot incremental via tabela de sinais
INSERT INTO debezium.signals (id, type, data)
VALUES ('snap-001', 'execute-snapshot',
        '{"data-collections": ["dbo.orders"]}');
```

> ℹ **O snapshot incremental é seguro para produção:** não bloqueia a tabela, pode ser pausado e retomado, e garante consistência via watermark interno. É a abordagem recomendada para tabelas grandes ou de alta concorrência.

### 10.4 Snapshot via Exportação Paralela

Para tabelas muito grandes (acima de centenas de GBs), realize o snapshot via exportação nativa do banco (`COPY TO` no PostgreSQL, `mysqldump` no MySQL) e importe diretamente na camada Bronze, registrando o LSN de início antes da exportação. O Debezium então retoma a partir desse LSN, evitando a leitura completa via JDBC.

---

## 11. Schema Evolution — DDL no Banco de Origem

Mudanças de DDL no banco de origem são um dos pontos mais críticos e menos documentados em pipelines CDC. Entender o comportamento de ponta a ponta — desde o banco até o consumidor — é essencial para evitar interrupções silenciosas.

### 11.1 O Que Acontece Quando um `ALTER TABLE` é Executado

O Debezium monitora eventos de DDL no transaction log e reage automaticamente, mas o comportamento varia conforme o banco e o tipo de mudança:

| Operação DDL | PostgreSQL | MySQL | SQL Server |
|---|---|---|---|
| `ADD COLUMN` | ✅ Detectado automaticamente | ✅ Detectado automaticamente | ✅ Detectado automaticamente |
| `DROP COLUMN` | ✅ Detectado | ✅ Detectado | ✅ Detectado |
| `RENAME COLUMN` | ⚠ Pode exigir reset do slot | ✅ Detectado | ⚠ Exige atenção |
| `CHANGE TYPE` | ⚠ Pode quebrar consumidores | ⚠ Pode quebrar consumidores | ⚠ Pode quebrar consumidores |

### 11.2 Política de Compatibilidade no Schema Registry

O Schema Registry é a peça central para lidar com evolução de schema de forma segura. A política de compatibilidade controla quais mudanças são permitidas sem quebrar consumidores existentes:

- **`BACKWARD` (recomendado como ponto de partida):** novos schemas podem ler dados escritos com schemas anteriores. Permite adicionar campos com valor default e remover campos opcionais. Consumidores antigos continuam funcionando.
- **`FORWARD`:** schemas antigos podem ler dados escritos com novos schemas. Permite adicionar campos opcionais.
- **`FULL`:** combinação de BACKWARD e FORWARD. A mais restritiva, mas oferece maior segurança em ambientes com múltiplos consumidores com versões diferentes.
- **`NONE`:** sem validação. Adequado apenas para desenvolvimento.

```json
// Configurar política no Schema Registry via API REST
PUT /config/dbserver1.dbo.orders-value
{
  "compatibility": "BACKWARD"
}
```

### 11.3 Ciclo de Vida de uma Mudança de Schema em Produção

O processo seguro para evoluir o schema em produção sem interrupção do pipeline:

```
1. Adicionar coluna com DEFAULT no banco de origem
   ALTER TABLE orders ADD COLUMN discount DECIMAL(10,2) DEFAULT 0.0;

2. Aguardar o Debezium detectar a mudança (eventos seguintes já incluirão o novo campo)

3. Verificar no Schema Registry que o novo schema foi registrado e aprovado
   GET /subjects/dbserver1.dbo.orders-value/versions/latest

4. Atualizar consumidores para lidar com o novo campo (compatibilidade BACKWARD
   garante que consumidores antigos ainda funcionem durante a transição)

5. Após todos os consumidores atualizados, remover suporte ao schema antigo se necessário
```

> ⚠ **`DROP COLUMN` e `RENAME COLUMN` são operações destrutivas.** Nenhuma política de compatibilidade permite essas mudanças sem coordenação explícita entre produtor e todos os consumidores. O processo correto é: (1) adicionar a nova coluna e começar a populá-la; (2) migrar todos os consumidores para usar a nova coluna; (3) só então remover ou renomear a coluna antiga — preferencialmente com janela de manutenção.

> ℹ **O Debezium oferece `column.exclude.list` e `column.mask.hash.*`** para filtrar ou mascarar colunas sensíveis diretamente no conector, antes da publicação no Kafka. Isso é particularmente útil para remover colunas que foram adicionadas ao banco mas não devem aparecer no pipeline de dados.

---

## 12. Segurança, PII e Conformidade

Em pipelines CDC, dados sensíveis do banco de origem chegam integralmente à camada Bronze por padrão — incluindo CPF, e-mail, dados de cartão de crédito e qualquer outra coluna com informação pessoal. Sem controles explícitos, o log de eventos se torna um vetor de vazamento de PII, com implicações diretas em LGPD, GDPR e PCI-DSS.

### 12.1 Mascaramento de Colunas PII no Conector

O Debezium oferece propriedades de configuração nativas para mascarar, hashear ou excluir colunas sensíveis **antes** da publicação no Kafka. **Não é necessário usar transforms**; basta configurar diretamente no conector:

```json
{
  "name": "connector",
  "config": {
    "connector.class": "io.debezium.connector.sqlserver.SqlServerConnector",
    "database.hostname": "...",
    "database.port": "...",
    "database.user": "...",
    "database.password": "...",
    "database.dbname": "...",
    "table.include.list": "dbo.orders",
    "database.server.name": "dbserver1",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.history.kafka.topic": "schema-changes.orders",

    "column.mask.hash.SHA-256.with.salt.mySalt": "dbo.customers.email,dbo.customers.cpf",
    "column.mask.with.12.chars": "dbo.payments.card_number,dbo.payments.cvv",
    "column.exclude.list": "dbo.customers.raw_password"
  }
}
```

> ✗ **Nunca confie apenas no controle de acesso ao tópico Kafka para proteção de PII.** O mascaramento deve ocorrer na origem — no conector — para garantir que dados sensíveis não sejam capturados no log, em snapshots ou em backups do Kafka.

### 12.2 Estratégias de Pseudonimização

Para casos em que é necessário correlacionar eventos de diferentes tabelas sem expor o identificador real:

- **Hash determinístico com salt:** `column.mask.hash.SHA-256.with.salt.{salt}` gera um pseudônimo estável que permite joins entre tabelas sem expor o valor original. Ideal para `customer_id`, `email`, `cpf`.
- **Tokenização:** substituição do valor real por um token gerado externamente (ex: Vault, AWS Tokenization). O mapeamento token → valor real fica em um sistema separado com acesso controlado.
- **Campos derivados via SMT (Single Message Transform):** use transforms customizadas para adicionar campos anonimizados ao envelope do evento enquanto remove o campo original.

### 12.3 Segurança em Trânsito e em Repouso

```json
// Configuração de TLS no conector Debezium (PostgreSQL como exemplo)
{
  "database.sslmode": "verify-full",
  "database.sslcert": "/secrets/client.crt",
  "database.sslkey": "/secrets/client.key",
  "database.sslrootcert": "/secrets/ca.crt"
}
```

- **TLS entre conector e banco:** obrigatório em produção. Configure `sslmode=verify-full` (PostgreSQL) ou `useSSL=true&requireSSL=true` (MySQL) para evitar downgrade de conexão.
- **TLS entre conector e Kafka:** configure `security.protocol=SSL` e os respectivos keystores e truststores no Kafka Connect worker.
- **Criptografia em repouso na Bronze:** utilize criptografia de disco gerenciada pelo cloud provider (SSE-S3, ADLS encryption) ou criptografia de coluna no formato de lake (Parquet column encryption).
- **Rotação de credenciais:** armazene senhas de banco e certificados em um secret manager (AWS Secrets Manager, HashiCorp Vault, Azure Key Vault) e configure o conector para ler as credenciais dinamicamente, sem hard-code em arquivos de configuração.

### 12.4 Controle de Acesso a Tópicos Kafka

Configure ACLs no Kafka para limitar quem pode produzir e consumir em cada tópico CDC:

```bash
# Permitir apenas o Debezium connector producer escrever no tópico
kafka-acls --add \
  --allow-principal User:debezium-connector \
  --operation Write \
  --topic dbserver1.dbo.orders

# Permitir apenas consumidores autorizados ler
kafka-acls --add \
  --allow-principal User:silver-pipeline \
  --operation Read \
  --topic dbserver1.dbo.orders \
  --group silver-consumer-group
```

> ℹ **Aplique o princípio do menor privilégio:** o usuário de banco configurado no Debezium deve ter apenas as permissões mínimas necessárias — leitura do log de replicação, acesso às tabelas alvo e, no PostgreSQL, `REPLICATION` role. Nunca use credenciais de administrador no conector.

### 12.5 Direito ao Esquecimento (Right to Erasure — LGPD/GDPR)

O direito ao esquecimento é um desafio específico de arquiteturas CDC porque a Bronze é imutável por design. As estratégias para lidar com requisições de exclusão sem comprometer o pipeline:

- **Crypto shredding:** criptografe os dados PII com uma chave por usuário **antes de gravar na Bronze** — não depois. Se os dados chegam em texto claro e a criptografia é aplicada em etapa posterior, há uma janela de exposição nos arquivos já escritos. O fluxo correto é: (1) criptografar no conector ou em um SMT antes da publicação no Kafka; (2) gravar apenas o dado já cifrado na Bronze; (3) destruir a chave quando o usuário solicitar a exclusão. Os eventos históricos permanecem, mas os dados se tornam irrecuperáveis.
- **Tombstone + reprocessamento:** publique um evento de exclusão no tópico CDC com os campos PII nulos ou tokenizados, e reconstrua a Silver a partir desse ponto.
- **Segregação de PII:** armazene PII em uma tabela separada referenciada por token na Bronze. A exclusão do mapeamento token → PII atende ao direito ao esquecimento sem alterar os eventos históricos.

---

## 13. Modos de Deploy do Debezium

O Debezium pode ser implantado de três formas distintas, cada uma com trade-offs de complexidade operacional, flexibilidade e destinos suportados.

### 13.1 Debezium Embedded no Kafka Connect (Padrão)

O modo mais utilizado em produção. O Debezium roda como um conector dentro de um cluster Kafka Connect, beneficiando-se do gerenciamento de offsets, distribuição de carga e monitoramento nativos do Connect.

```bash
# Deploy via API REST do Kafka Connect
curl -X POST http://kafka-connect:8083/connectors \
  -H 'Content-Type: application/json' \
  -d @debezium-connector-config.json
```

**Indicado para:** times com Kafka Connect já operacional, múltiplos conectores, necessidade de escalabilidade horizontal.

### 13.2 Debezium Server (Standalone)

O **Debezium Server** é uma aplicação standalone que não requer Kafka Connect. Ele lê o transaction log do banco e publica eventos diretamente em diferentes destinos sem passar pelo Kafka.

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

# Outros destinos suportados: kafka, pubsub, redis, http, rabbitmq, nats
```

**Destinos suportados pelo Debezium Server:**

| Destino | Tipo | Caso de Uso |
|---|---|---|
| Kafka | Streaming | Pipeline completo com Kafka |
| Amazon Kinesis | Streaming gerenciado | Ambientes AWS sem Kafka |
| Google Pub/Sub | Streaming gerenciado | Ambientes GCP |
| Azure Event Hubs | Streaming gerenciado | Ambientes Azure |
| Redis Streams | In-memory | Latência ultra-baixa |
| HTTP/Webhook | Genérico | Integrações customizadas |
| RabbitMQ | Message broker | Ambientes com RabbitMQ |

**Indicado para:** times que não querem ou não podem operar Kafka, ambientes serverless, destinos cloud nativos, ou casos onde o volume não justifica um cluster Kafka completo.

### 13.3 Debezium Embedded Library

Para casos onde o CDC deve ser executado dentro de uma aplicação JVM existente, o Debezium oferece uma biblioteca Java que pode ser incorporada diretamente no código da aplicação.

```java
// Debezium Embedded — integração direta em aplicação Java/Kotlin
DebeziumEngine<ChangeEvent<String, String>> engine = DebeziumEngine
    .create(Json.class)
    .using(props)
    .notifying(record -> {
        // processar evento CDC diretamente na aplicação
        processChangeEvent(record);
    })
    .build();

ExecutorService executor = Executors.newSingleThreadExecutor();
executor.execute(engine);
```

**Indicado para:** microserviços que precisam reagir a mudanças do próprio banco sem infraestrutura de streaming, casos de uso de cache invalidation ou event sourcing em serviços específicos.

> ℹ **Comparativo rápido:** Kafka Connect oferece o maior ecossistema e escalabilidade. Debezium Server simplifica o deploy quando Kafka não é necessário ou desejável. Debezium Embedded é a opção mais simples para integração direta em aplicações JVM sem infraestrutura adicional.

---

## 14. Testando Pipelines CDC

Pipelines CDC são notoriamente difíceis de testar porque envolvem múltiplos sistemas externos (banco de dados, Kafka, storage) e dependem do comportamento de transaction logs. Sem uma estratégia de testes, regressões silenciosas — como o bug de `after.id` em DELETEs — passam para produção sem detecção.

### 14.1 Testes de Unidade — Validação do Schema de Eventos

O primeiro nível de testes não requer nenhuma infraestrutura externa. Valide a lógica de transformação com payloads JSON estáticos que representam cada tipo de evento (`c`, `u`, `d`, `r`):

```python
import pytest
from pyspark.sql import SparkSession
from delta.tables import DeltaTable

@pytest.fixture(scope='session')
def spark():
    return SparkSession.builder \
        .config('spark.jars.packages', 'io.delta:delta-core_2.12:2.4.0') \
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
    assert result.first()['entity_id'] == 42   # deve usar before.id

def test_insert_event_id_extraction(spark):
    """Garante que a extração de ID usa after.id para INSERTs."""
    df = spark.createDataFrame([INSERT_EVENT])
    result = df.selectExpr(
        "CASE WHEN op = 'd' THEN before.id ELSE after.id END AS entity_id"
    )
    assert result.first()['entity_id'] == 43   # deve usar after.id
```

### 14.2 Testes de Integração com Testcontainers

Para testar o pipeline de ponta a ponta localmente, use **Testcontainers** para subir PostgreSQL, Kafka e Debezium em containers Docker durante o teste:

```python
# pytest com testcontainers-python
import pytest
import json
import psycopg2
from kafka import KafkaConsumer
from testcontainers.postgres import PostgresContainer
from testcontainers.kafka import KafkaContainer

# Nota: esses testes assumem que um worker Kafka Connect com Debezium
# está configurado para apontar para os containers de banco e Kafka.
# Em um cenário real, use docker-compose para subir o Connect também.

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
    """Fixture reutilizável para criar um KafkaConsumer apontando para o container."""
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
    """
    Testa que um INSERT no banco gera o evento correto no tópico Kafka.
    O Debezium Connect é iniciado apontando para os containers.
    """
    conn = psycopg2.connect(postgres.get_connection_url())
    conn.cursor().execute("INSERT INTO orders (id, status) VALUES (1, 'open')")
    conn.commit()

    message = next(kafka_consumer)
    payload = message.value['payload']

    assert payload['op'] == 'c'
    assert payload['after']['id'] == 1
    assert payload['after']['status'] == 'open'
    assert payload['before'] is None

def test_delete_event_uses_before_id(postgres, kafka_consumer):
    """
    Garante que eventos de DELETE contêm before.id e after=null.
    Esse é o caso que falha silenciosamente se o merge usar after.id.
    """
    conn = psycopg2.connect(postgres.get_connection_url())
    conn.cursor().execute("DELETE FROM orders WHERE id = 1")
    conn.commit()

    message = next(kafka_consumer)
    payload = message.value['payload']

    assert payload['op'] == 'd'
    assert payload['before']['id'] == 1
    assert payload['after'] is None          # after DEVE ser null
```

### 14.3 Testes de Regressão — Cenários Críticos

Os cenários abaixo devem fazer parte obrigatória da suíte de testes de qualquer pipeline CDC. São os casos que mais frequentemente escapam para produção sem testes:

| Cenário | O que testar | Por que é crítico |
|---|---|---|
| DELETE event | `after` é `null`; merge usa `before.id` | Bug mais comum em merges CDC |
| Replay de eventos | Reprocessar os mesmos offsets produz o mesmo estado | Garantia de idempotência |
| Schema change (`ADD COLUMN`) | Novo campo aparece no evento; consumidores não quebram | Schema evolution |
| Conector reiniciado após pausa | Nenhum evento é perdido ou duplicado além do esperado | At-least-once delivery |
| Slot inativo (PostgreSQL) | Alertas disparam antes de `lag_bytes` atingir threshold crítico | Proteção contra disk full |
| Tombstone após DELETE | Consumidores com log compaction recebem e ignoram tombstones corretamente | Compactação de tópicos |
| Deduplicação MySQL | Offset composto `file+pos` deduplica corretamente no rollover de arquivo | Idempotência multi-banco |
| Deduplicação MongoDB | Resume token (`source.resume_token`) usado como chave única | Idempotência no MongoDB |

> ℹ **Automatize os testes de regressão em CI.** Testcontainers permite rodar o banco, o Kafka e o Debezium em pipelines de CI/CD sem infraestrutura dedicada. Um pipeline CDC sem testes automatizados é um pipeline que vai quebrar silenciosamente em produção.

---

## 15. Desafios Práticos e Como Mitigá-los

### 15.1 Replication Slot Bloat — PostgreSQL

O maior risco operacional em CDC com PostgreSQL. Um slot inativo impede o PostgreSQL de descartar segmentos WAL, causando crescimento ilimitado do disco.

- **Monitorar continuamente:** configure alertas quando `pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)` ultrapassar o threshold.
- **Safety net (PostgreSQL 13+):** defina `max_slot_wal_keep_size` para limitar o crescimento máximo de WAL retido.
- **Slot inativo prolongado:** drope o slot imediatamente e recrie via snapshot incremental do Debezium.

### 15.2 Schema Evolution

Adições de colunas no banco de origem precisam ser propagadas de forma compatível para downstream. Utilize Schema Registry com política de compatibilidade `BACKWARD` ou `FULL` para garantir que consumidores antigos continuem funcionando após mudanças de schema. Remoção e renomeação de colunas exigem estratégia de migração coordenada entre produtor e todos os consumidores — consulte a seção 11 para o ciclo de vida completo.

### 15.3 Consumer Lag e Monitoramento

O consumer lag do Kafka indica quanto os consumidores estão atrasados em relação ao produtor CDC. Configure alertas em Prometheus/Grafana para lag acima dos thresholds definidos pelo SLA. Métricas-chave: `kafka_consumer_lag` por grupo e tópico, `debezium_metrics_MilliSecondsBehindSource`, e crescimento de `lag_bytes` nos replication slots.

### 15.4 Tombstones e Log Compaction

Ao configurar tópicos CDC com log compaction habilitado (limpeza por chave), certifique-se de que os consumidores estejam preparados para receber tombstones (mensagens com valor `null`) que o Debezium publica após eventos de DELETE. Ignorar tombstones em tópicos compactados pode levar a dados desatualizados nos consumidores.

### 15.5 Tabelas sem Chave Primária

Tabelas sem PK representam um caso especial que requer atenção. No PostgreSQL, é possível usar `REPLICA IDENTITY FULL` para incluir todos os campos no evento de DELETE, mas isso aumenta significativamente o volume do WAL. No MySQL, eventos de UPDATE e DELETE em tabelas sem PK são publicados com todos os campos no `before`, mas sem identificador único — o merge na Silver exige uma chave composta artificial ou hash dos campos.

```sql
-- PostgreSQL: forçar captura completa em tabela sem PK
ALTER TABLE legacy_table REPLICA IDENTITY FULL;
```

> ⚠ **Tabelas sem PK são um sinal de problema de modelagem.** Sempre que possível, adicione uma coluna de PK antes de configurar o CDC — mesmo que seja um `SERIAL` ou `UUID` gerado automaticamente.

---

## 16. CDC em Arquiteturas de Data Lake

Os formatos open source de tabelas para data lakes — Delta Lake, Apache Iceberg e Apache Hudi — oferecem suporte nativo a operações de upsert e delete, tornando-os opções ideais para a camada Silver de pipelines CDC.

### 16.1 Comparativo dos Formatos

- **Delta Lake (Databricks/OSS):** `MERGE INTO` nativo com suporte a schema evolution e time travel. Maior adoção em ecossistema Spark. Compactação via `OPTIMIZE` e `VACUUM`.
- **Apache Iceberg:** suporte a múltiplos engines (Spark, Flink, Trino, Hive). MERGE via merge-on-read ou copy-on-write configurável por tabela. Excelente para cenários multi-engine.
- **Apache Hudi:** nativamente orientado a CDC com suporte a Merge On Read (MOR) e Copy On Write (COW). Recursos built-in de incremental queries e bootstrap a partir de datasets existentes.

### 16.2 Hudi — Design Orientado a CDC

O Apache Hudi suporta dois modos de escrita com trade-offs distintos:

- **Copy On Write (COW):** cada atualização reescreve o arquivo Parquet completo. Leituras rápidas, escritas mais custosas. Recomendado para cargas com mais leitura do que escrita.
- **Merge On Read (MOR):** atualizações são escritas em logs delta e mescladas na leitura. Ingestão de baixíssima latência, leituras ligeiramente mais custosas. Recomendado para alta frequência de CDC.

> ℹ **Para pipelines CDC com alta frequência de UPDATEs, Hudi MOR é frequentemente a escolha mais eficiente.** Para casos de uso analítico pesado com poucas atualizações, Delta Lake COW ou Iceberg são mais indicados.

> ℹ **Hudi MOR — Compactação obrigatória:** no modo Merge On Read, as atualizações são gravadas em arquivos de log Avro e mescladas dinamicamente na leitura. Com o tempo, o acúmulo de logs degrada a performance de leitura. Existem dois modos de compactação:
>
> - **Inline** (`hoodie.compact.inline=true`): executada junto com cada commit, configurável por número de commits via `hoodie.compact.inline.max.delta.commits`. **Atenção:** o commit fica bloqueado até a compactação terminar — em tabelas grandes, isso pode causar latência de ingestão perceptível. Evite em produção com alta frequência de escrita.
> - **Assíncrona** (recomendada para produção, via `HoodieCompactionConfig`): mescla logs em Parquet em background sem bloquear a ingestão. Agende por número de commits ou por intervalo de tempo conforme o SLA de leitura. Permite que escritas e leituras continuem enquanto a compactação ocorre em paralelo.

### 16.3 Time Travel e Auditoria

Todos os três formatos suportam time travel — consulta de versões históricas da tabela por timestamp ou versão.

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

## 17. CDC em Ambientes Cloud Gerenciados

| Serviço | Provider | Bancos Suportados | Destinos Nativos | Observação | Limites/Preços Típicos |
|---|---|---|---|---|---|
| AWS DMS | Amazon | Oracle, SQL Server, MySQL, PostgreSQL, MongoDB | S3, Redshift, Kinesis | Snapshot + CDC contínuo; custo por instância de replicação | Instância por hora + transferência; latência tipicamente de segundos, mas pode exceder minutos com Full LOB mode ou cargas elevadas |
| Confluent Cloud | Confluent | PostgreSQL, MySQL, SQL Server, MongoDB, Oracle | Kafka nativo | Debezium gerenciado; integração Kafka | Custo por volume de dados + conectores; latência < 1s |
| Azure Data Factory | Microsoft | SQL Server/Azure SQL, PostgreSQL, MySQL, Oracle | ADLS, Synapse, Azure SQL | SQL Server/Azure SQL: Change Tracking; demais: log-based | Preço por execução de pipeline; latência variável |
| Google Datastream | Google | Oracle, MySQL, PostgreSQL | BigQuery, GCS, Spanner | Totalmente serverless; integração nativa com BigQuery | Custo por volume de dados; latência < 10s |
| Fivetran / Airbyte | SaaS / OSS | 50+ conectores | Warehouse, lakes | Airbyte open source; Fivetran gerenciado | Por volume de linhas (Fivetran); auto-gerenciado gratuito |

> ⚠ **AWS DMS e latência:** o valor de latência "segundos" é válido para configurações padrão com tipos de dados simples. Com **Full LOB mode** habilitado (necessário para colunas TEXT, BLOB, CLOB) ou em cargas com alto volume de escritas, a latência pode subir para vários minutos. Avalie o modo de LOB e o tamanho de chunk antes de definir SLAs.

### 17.1 Critérios de Escolha

- **Controle e customização:** Debezium auto-gerenciado oferece máximo controle sobre configurações, transforms e schema evolution. Indicado para times com experiência em Kafka.
- **Redução de overhead operacional:** Confluent Cloud (Debezium gerenciado) ou AWS DMS são indicados para times que preferem delegar o gerenciamento da infraestrutura de CDC.
- **Integração nativa com cloud:** Google Datastream integra nativamente com BigQuery; AWS DMS com Kinesis e Redshift; Azure Data Factory com Synapse Analytics.
- **Custo:** serviços gerenciados eliminam o custo de operação, mas podem ser mais caros em volume elevado. Faça a análise de TCO considerando custo de engenharia de infraestrutura vs custo do serviço.

> ℹ **Airbyte** é uma alternativa open source para times que querem conectores gerenciados sem custo de SaaS, com suporte a 50+ fontes e destinos. A versão Cloud está disponível como serviço gerenciado.

---

## 18. Dimensionamento e Custos

### 18.1 Estimativa de Throughput

O volume de dados em um pipeline CDC é função da taxa de mudanças no banco de origem. Para dimensionar Kafka e os conectores:

- **Número de partições:** idealmente igual ao número de consumidores paralelos. Debezium particiona por chave primária; partições em excesso aumentam overhead.
- **Retenção no Kafka:** defina com base na janela de reprocessamento necessária. Para retenção longa, considere armazenamento em camadas (tiered storage) se disponível.
- **Tamanho médio do evento:** some os campos before/after. Um evento típico pode ter de 200 bytes a vários KB.
- **Throughput esperado:** eventos por segundo × tamanho médio. Ex: 10.000 eventos/s × 1 KB = 10 MB/s ≈ 86 GB/dia.

### 18.2 Custos Operacionais

- **Kafka:** custo de brokers (memória, disco, rede). Em nuvem, use managed Kafka (Confluent, MSK, Event Hubs) para reduzir overhead.
- **Armazenamento na Bronze:** dados imutáveis ocupam espaço. Use compressão (Parquet, ORC) e políticas de lifecycle (ex: mover para camada fria após N dias).
- **Processamento:** Spark estruturado, Flink ou engines serverless (Databricks, EMR) para transformações.
- **Banco de origem:** impacto no IO devido à leitura do log; monitore e dimensione storage adequadamente.

### 18.3 Exemplo Numérico

Suponha uma tabela de pedidos com 1 milhão de atualizações por dia, cada evento com 1 KB.  
- Throughput médio: 1.000.000 / 86400 ≈ 11,6 eventos/s ≈ 11,6 KB/s (insignificante).  
- Picos podem ser 10x maiores.  
- Com retenção de 7 dias no Kafka, armazenamento ≈ 11,6 KB/s × 604800 s ≈ 7 GB.  
- Se houver 10 tabelas similares, 70 GB de armazenamento no Kafka.  
- Custo mensal aproximado (AWS MSK, 3 brokers m5.large): ~ $600 + armazenamento EBS.

> ℹ **Recomendação:** comece com uma estimativa conservadora e monitore o crescimento real nos primeiros meses. Ajuste retenção e número de partições conforme necessidade.

---

## 19. Monitoramento e Alertas

Um pipeline CDC maduro exige monitoramento contínuo. Configure métricas e alertas nos seguintes níveis:

### 19.1 Métricas do Debezium (via JMX)

Habilitar JMX no Kafka Connect expõe métricas como:

- `debezium_metrics_MilliSecondsBehindSource`: lag entre o timestamp do evento no banco e o momento em que foi publicado no Kafka.
- `debezium_metrics_TotalNumberOfEventsSeen`: total de eventos processados.
- `debezium_metrics_QueueRemainingCapacity`: capacidade restante da fila interna.

### 19.2 Métricas do Kafka

- **Consumer lag por grupo:** `kafka_consumer_lag` (Prometheus + kafka-exporter). Alerte se lag > threshold (ex: 10 minutos).
- **Taxa de produção/consumo:** `kafka_server_BrokerTopicMetrics_MessagesInPerSec`, `BytesInPerSec`.
- **Erros de conectores:** via API REST do Kafka Connect.

### 19.3 Métricas do Banco de Origem

- **PostgreSQL:** tamanho do WAL retido por replication slot (`pg_replication_slots`). Alerte se `lag_bytes` > 100 GB ou tempo > 1 hora.
- **MySQL:** espaço ocupado pelos binlogs e arquivo mais antigo ainda necessário. Monitore `expire_logs_days` e `binlog_expire_logs_seconds`.
- **SQL Server:** tamanho da tabela de captura do CDC e latência de leitura.

### 19.4 Métricas da Camada Bronze/Silver

- **Taxa de ingestão:** registros por segundo no Delta Lake.
- **Deduplicação:** número de duplicatas detectadas (deve ser próximo de zero em condições normais).
- **Erros de merge:** falhas no `MERGE` (ex: chave duplicada, schema incompatível).

### 19.5 Alertas Recomendados

| Alerta | Condição | Ação |
|---|---|---|
| Replication slot lag | `lag_bytes > 50 GB` ou tempo > 30 min | Investigar conector; dropar slot se necessário |
| Consumer lag | `lag > 10 min` (ou conforme SLA) | Escalar consumidores ou verificar gargalos |
| Conector parado | Conector não está rodando (API) | Reiniciar via API / alertar equipe |
| Erros de DDL | Evento de DDL não processado | Verificar Schema Registry e atualizar schemas |
| Queda de throughput | Redução súbita > 50% | Verificar saúde do banco e conectores |
| Aproximação do limite de retenção | Binlog/WAL próximo da expiração | Aumentar retenção ou investigar consumo lento |

---

## 20. Testes de Resiliência

Além dos testes de integração, é fundamental validar o comportamento do pipeline sob falhas. Use Testcontainers ou ambientes de staging para simular:

### 20.1 Cenários de Falha

1. **Banco de origem fica indisponível:** parar o container do banco por alguns minutos e depois reiniciar. Verificar se o conector retoma do offset correto e se o lag é recuperado.
2. **Kafka fora do ar:** simular falha nos brokers. O conector deve tentar reconectar com backoff e, ao restabelecer, continuar de onde parou.
3. **Reinicialização do conector:** matar o processo do Kafka Connect e verificar se os offsets são restaurados e nenhum evento é perdido.
4. **Corrupção de offset:** forçar um offset inválido no tópico de offsets do Kafka Connect e verificar se o conector consegue se recuperar via snapshot.
5. **Sobrecarga de mensagens:** gerar picos de inserções/atualizações e observar se o pipeline consegue acompanhar ou se o lag aumenta de forma controlada.
6. **Deduplicação:** enviar manualmente mensagens duplicadas (mesmo offset) e confirmar que a Silver não as aplica duas vezes.

### 20.2 Ferramentas

- **Testcontainers** para orquestrar banco, Kafka e Debezium.
- **Apache JMeter** ou **k6** para gerar carga no banco.
- **Chaos Mesh** ou **Gremlin** para injeção de falhas em Kubernetes.

> ℹ **Automatize esses testes em um ambiente de staging antes de promover mudanças para produção.** A resiliência não é um evento único, mas uma qualidade que deve ser continuamente validada.

---

## 21. Troubleshooting Comum

### 21.1 Conector não inicia

- Verifique se as configurações de banco estão corretas (host, porta, credenciais).
- Confirme que o banco está com log habilitado (`wal_level=logical`, `binlog_format=ROW`).
- No PostgreSQL, certifique-se de que o replication slot não existe previamente (ou use `database.history.skip.uncreate`).
- No Kafka Connect, verifique se o worker pode acessar os tópicos de offset e histórico.

### 21.2 Eventos não chegam ao Kafka

- Verifique se o conector está em execução (API `/connectors/status`).
- Monitore o lag no banco; se estiver alto, o conector pode estar com problemas de rede ou desempenho.
- Confira se o tópico de destino foi criado e tem partições suficientes.

### 21.3 Eventos de DELETE não são aplicados na Silver

- Provavelmente o merge está usando `after.id` diretamente. Aplique a correção com `CASE WHEN` e garanta que `whenMatchedDelete` seja declarado antes.

### 21.4 Duplicatas na Silver

- Verifique se a deduplicação por offset está sendo feita corretamente. Para MySQL, lembre-se de usar `source.file + ':' + source.pos`.
- Confirme que o `_offset_key` está sendo armazenado e usado como critério de deduplicação.

### 21.5 Schema incompatível no Schema Registry

- Verifique a política de compatibilidade. Se adicionou uma coluna sem default, consumidores antigos podem quebrar.
- Use `BACKWARD` e adicione colunas com `DEFAULT` no banco.

### 21.6 Disk Full no PostgreSQL devido ao WAL

- O replication slot está parado. Drope o slot ou aumente `max_slot_wal_keep_size`.
- Configure alertas para lag do slot.

---

## 22. Conclusão

CDC log-based combinado com Kafka e uma arquitetura em camadas (Bronze → Silver → Gold) é o padrão de referência para pipelines de dados modernos que exigem baixa latência, confiabilidade e auditoria completa.

Os pilares que sustentam um pipeline CDC maduro em produção:

- **Imutabilidade da Bronze:** fonte única de verdade e ponto de replay para qualquer reprocessamento futuro.
- **Debezium com particionamento correto:** chave primária como message key Kafka garante ordering por entidade e paralelismo seguro.
- **Merge idempotente na Silver:** `whenMatchedDelete` antes de `whenMatchedUpdate`, deduplicação por `_offset_key` adequada ao banco de origem, e armazenamento do offset como coluna de auditoria.
- **Snapshot incremental:** eliminação do lock global via chunk-based snapshot com watermark interno.
- **Schema evolution gerenciada:** Schema Registry com política `BACKWARD` ou `FULL`, e ciclo de vida explícito para operações destrutivas de DDL.
- **Segurança e PII desde a origem:** mascaramento de colunas sensíveis no conector, crypto shredding aplicado antes da gravação na Bronze, TLS em trânsito, criptografia em repouso e estratégia de direito ao esquecimento.
- **Monitoramento proativo:** replication slot lag, consumer lag, espaço de Binlog e alertas de schema incompatível.
- **Testes automatizados e de resiliência:** suíte de regressão com Testcontainers cobrindo cenários críticos e falhas de infraestrutura.
- **Escolha consciente do formato de lake:** Delta Lake, Iceberg ou Hudi conforme padrão de carga e engine de processamento — com atenção ao modo de compactação do Hudi MOR em produção.
- **Dimensionamento e custos controlados:** estimativas realistas e monitoramento contínuo para ajustes.

O próximo nível após o CDC maduro é a convergência com **event sourcing** na camada de aplicação — onde os próprios sistemas produtores emitem eventos como cidadãos de primeira classe, eliminando a dependência do transaction log como proxy. Mas enquanto a maioria dos sistemas produtores ainda é construída sobre bancos relacionais tradicionais, CDC log-based com Debezium continua sendo o caminho mais pragmático e robusto para construir plataformas de dados em tempo real.

---

## Glossário

| Termo | Significado |
|-------|-------------|
| **WAL** | Write-Ahead Log (PostgreSQL) |
| **Binlog** | Binary Log (MySQL) |
| **LSN** | Log Sequence Number (PostgreSQL, SQL Server) |
| **Resume Token** | Identificador de checkpoint no MongoDB Change Streams |
| **Replication Slot** | Mecanismo do PostgreSQL que retém WAL até ser consumido |
| **Offset** | Posição no log que permite retomada exata |
| **Schema Registry** | Serviço que armazena versões de schemas Avro/JSON/Protobuf |
| **PII** | Personally Identifiable Information |
| **CDC** | Change Data Capture |
| **Debezium** | Conector open source para CDC |
| **Kafka Connect** | Framework para conectar Kafka a sistemas externos |
| **Delta Lake / Iceberg / Hudi** | Formatos de tabela para data lakes com suporte a upsert/delete |
| **Full LOB mode** | Modo do AWS DMS para replicar colunas de objeto grande (TEXT, BLOB, CLOB) |
| **dropDuplicatesWithinWatermark** | Operação Spark 3.5+ para deduplicação com janela temporal limitada, evitando crescimento de estado ilimitado |

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
- [Azure Data Factory](https://azure.microsoft.com/en-us/services/data-factory/)
- [Spark dropDuplicatesWithinWatermark](https://spark.apache.org/docs/3.5.0/structured-streaming-programming-guide.html)

---

> **Pipeline CDC maduro = log-based + Debezium + Kafka + Bronze imutável + merge idempotente com `CASE WHEN` + `whenMatchedDelete` primeiro + deduplicação por offset composto + `dropDuplicatesWithinWatermark` em streams longos + crypto shredding antes da Bronze + schema evolution gerenciada + PII mascarado na origem + testes automatizados + monitoramento contínuo + resiliência validada + custo controlado.**
