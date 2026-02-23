90% dos pipelines de CDC falham de forma silenciosa. Você confia no seu MERGE? 🚨

Implementar Change Data Capture (CDC) log-based com Debezium e Kafka parece lindo na arquitetura. Mas em produção, a realidade bate à porta com força:

❌ Eventos de DELETE ignorados porque o código tentou ler um after.id nulo.
❌ Deduplicação ingênua com dropDuplicates no PySpark descartando dados vitais de transações em lote.
❌ Bancos PostgreSQL caindo por disk full devido a replication slots esquecidos.

Decidi documentar como resolver esses (e muitos outros) problemas estruturais. Escrevi o artigo "Change Data Capture — Streaming Edition", um guia prático focado em quem precisa manter dados consistentes e arquiteturas resilientes em produção.

Lá eu abordo:

O erro fatal da deduplicação na Camada Silver e como corrigir com Window Functions.

Semânticas de idempotência (Exactly-once vs At-least-once).

Gestão de Schema Evolution e PII direto na origem.

Exemplos táticos em PostgreSQL, MySQL e SQL Server.

Quer ler o material completo e blindar seus pipelines?
👇 Comente "CDC" aqui embaixo e eu te envio o link direto na sua DM.

#DataEngineering #CDC #Kafka #Debezium #ApacheSpark #DataArchitecture
