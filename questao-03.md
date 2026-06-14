# Questão 03 — Diagnóstico de Lentidão em Produção

> **Cenário:** Após uma nova versão do sistema entrar em produção, usuários começam a reportar lentidão em diversas funcionalidades. Como DBA, descreva seu processo de investigação desde os primeiros sinais do problema até a identificação da causa raiz. Explique quais informações coletaria, quais ferramentas utilizaria e como diferenciaria problemas de banco, aplicação ou infraestrutura.

---

## Premissa: O Deploy é o Principal Suspeito

Quando a lentidão começa imediatamente após um deploy, o contexto já filtra as hipóteses. Causas aleatórias (hardware, crescimento gradual de dados) são improvàveis. O foco vai para **o que mudou**: código novo, migrations de schema, alterações de configuração ou aumento de volume de queries.

A investigação segue uma ordem de camadas: **Infraestrutura → Banco de Dados → Aplicação**, eliminando cada nível antes de avançar.

---

## Fase 1 — Triagem Inicial (0 a 5 minutos)

### 1.1 Estabelecer o escopo do problema

Antes de qualquer ferramenta, responder:

- A lentidão é **generalizada** (todo o sistema) ou **localizada** (funcionalidades específicas)?
- Quando exatamente começou? Coincide com o horário do deploy?
- Há erros explícitos (timeouts, connection refused) ou apenas lentidão?
- O problema é **crescente** (piorando) ou **estável** (ruim mas constante)?

Respostas generalizada + início no deploy + crescente = suspeita forte de **lock chain** ou **pool de conexões esgotado**.
Respostas localizada + estável = suspeita de **query específica sem índice** após migration.

### 1.2 Verificação rápida do banco

```sql
-- Quantas conexões ativas agora?
SELECT count(*), state
FROM pg_stat_activity
GROUP BY state
ORDER BY count DESC;

-- Há algum processo rodando há muito tempo?
SELECT pid, usename, state, now() - query_start AS duration,
       LEFT(query, 120) AS query_preview
FROM pg_stat_activity
WHERE state != 'idle'
  AND query_start < now() - interval '30 seconds'
ORDER BY duration DESC;

-- Há locks bloqueando outros processos?
SELECT count(*) AS processos_bloqueados
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

Se `processos_bloqueados > 0` → **lock chain** é a causa imediata. Ir direto para seção 2.2.
Se conexões próximas de `max_connections` → **pool esgotado**. Ir para seção 3.2.

---

## Fase 2 — Diagnóstico na Camada de Banco

### 2.1 Identificar as queries mais lentas

```sql
-- Top queries por tempo total de execução (acumulado desde o deploy)
SELECT
    LEFT(query, 100) AS query,
    calls,
    ROUND(total_exec_time::numeric, 2) AS total_ms,
    ROUND(mean_exec_time::numeric, 2) AS media_ms,
    ROUND(stddev_exec_time::numeric, 2) AS desvio_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Queries com ALTA variação (stddev alto = comportamento instável)
SELECT
    LEFT(query, 100) AS query,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS media_ms,
    ROUND(stddev_exec_time::numeric, 2) AS desvio_ms,
    ROUND((stddev_exec_time / NULLIF(mean_exec_time, 0))::numeric, 2) AS coef_variacao
FROM pg_stat_statements
WHERE calls > 10
ORDER BY coef_variacao DESC
LIMIT 20;
```

> **`pg_stat_statements`** acumula dados desde o último `pg_stat_statements_reset()`. Resetar antes do próximo deploy de teste para ter dados limpos.

### 2.2 Investigar Lock Chains

```sql
-- Identificar cadeia completa de bloqueios
WITH RECURSIVE lock_tree AS (
    -- Processos bloqueados
    SELECT
        blocked.pid AS blocked_pid,
        blocked.usename AS blocked_user,
        blocking.pid AS blocking_pid,
        blocking.usename AS blocking_user,
        blocked.query AS blocked_query,
        blocking.query AS blocking_query,
        blocking.state AS blocking_state,
        now() - blocking.query_start AS blocking_duration
    FROM pg_stat_activity blocked
    JOIN pg_stat_activity blocking
        ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
)
SELECT * FROM lock_tree
ORDER BY blocking_duration DESC;
```

**O que procurar:**
- `blocking_state = 'idle in transaction'` → processo abriu transação e parou. Causa mais comum após deploy: código novo com transações abertas por longo tempo (ex: chamada de API dentro de um `BEGIN`)
- Um único `blocking_pid` causando dezenas de `blocked_pid` → lock em cascade, eliminar o bloqueador resolve

```sql
-- Se identificado processo bloqueante problemático (confirmar antes de terminar)
SELECT pid, usename, application_name, state, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE pid = <blocking_pid>;

-- Terminar conexão problemática (apenas após confirmação)
SELECT pg_terminate_backend(<blocking_pid>);
```

### 2.3 Verificar Plano de Execução de Queries Suspeitas

Após identificar a query lenta via `pg_stat_statements`:

```sql
-- Analisar o plano de execução com métricas reais
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders o
JOIN order_items oi ON oi.order_id = o.id
WHERE o.customer_id = 12345
  AND o.status = 'pending';
```

**Sinais de alerta no plano:**
- `Seq Scan` em tabela grande → falta de índice ou estatísticas desatualizadas
- `Rows Removed by Filter` muito alto → índice ineficiente para o filtro
- `Buffers: hit=0 read=XXXXX` → dados não estão em cache, pressão de I/O
- `Nested Loop` com muitas iterações → possível N+1 vindo da aplicação

### 2.4 Verificar Migrations do Deploy

```sql
-- Verificar se algum índice foi removido ou criado sem CONCURRENTLY
SELECT schemaname, tablename, indexname, indexdef
FROM pg_indexes
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, tablename;

-- Comparar com lista de índices esperada (do ambiente de homologação)
-- Índice faltante após migration é uma das causas mais comuns de lentidão pós-deploy

-- Verificar se há índices inválidos (criados com erro)
SELECT schemaname, tablename, indexname, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
JOIN pg_index ON pg_index.indexrelid = pg_stat_user_indexes.indexrelid
WHERE NOT pg_index.indisvalid;
```

### 2.5 Estatísticas Desatualizadas (após migrations com muitos dados)

```sql
-- Verificar quando foi o último ANALYZE nas tabelas afetadas
SELECT schemaname, relname,
       n_live_tup,
       last_analyze,
       last_autoanalyze
FROM pg_stat_user_tables
WHERE relname IN ('orders', 'order_items', 'customers') -- tabelas suspeitas
ORDER BY last_analyze DESC NULLS LAST;

-- Se desatualizadas, forçar ANALYZE nas tabelas críticas
ANALYZE VERBOSE orders;
ANALYZE VERBOSE order_items;
```

---

## Fase 3 — Diagnóstico na Camada de Infraestrutura

### 3.1 Métricas de SO (executar no servidor)

```bash
# CPU: verificar se há processo consumindo 100%
top -bn1 | head -20
# ou: verificar CPU steal (em ambientes cloud — indica throttling da VM)
vmstat 1 5

# Disco: verificar I/O wait alto
iostat -x 1 5
# iowait > 20% = gargalo de disco

# Memória: verificar se há swap em uso (crítico para PostgreSQL)
free -h
vmstat -s | grep -E "swap|used"

# PostgreSQL usa shared_buffers para cache. Se o SO está sofrendo de
# pressão de memória, o cache de páginas do kernel (page cache) diminui,
# causando mais leituras físicas de disco → lentidão
```

**Sinais de problema de infra:**
- `iowait > 15%` consistentemente → gargalo de disco (IOPS insuficiente)
- Swap em uso com PostgreSQL → `shared_buffers` ou `work_mem` mal configurados
- CPU steal alto (> 5%) → a VM está sendo throttled pelo hypervisor (comum em cloud)

### 3.2 Conexões e Pool

```sql
-- Verificar utilização de max_connections
SELECT
    max_conn,
    used_conn,
    ROUND(used_conn::numeric / max_conn * 100, 1) AS utilizacao_pct
FROM
    (SELECT setting::int AS max_conn FROM pg_settings WHERE name = 'max_connections') mc,
    (SELECT count(*) AS used_conn FROM pg_stat_activity) uc;

-- Verificar conexões por aplicação (detectar aplicação abrindo conexões demais)
SELECT application_name, count(*), array_agg(DISTINCT state) AS estados
FROM pg_stat_activity
GROUP BY application_name
ORDER BY count DESC;
```

Se uma aplicação específica (provavelmente a que recebeu o deploy) está com muito mais conexões que o esperado → código novo com connection leak ou sem connection pool configurado corretamente.

---

## Fase 4 — Diferenciação: Banco vs Aplicação vs Infra

| Sintoma | Camada mais provável |
|---|---|
| Seq Scan em tabela grande após migration | **Banco** — índice removido/faltante |
| `idle in transaction` em muitos processos | **Aplicação** — transação não encerrada corretamente |
| Queries simples lentas, plano normal | **Infra** — I/O wait, CPU steal |
| Muitas conexões abertas por uma aplicação | **Aplicação** — connection pool mal configurado |
| Degradação progressiva durante o dia | **Banco** — bloat, autovacuum não dando conta |
| Lentidão em apenas 1-2 funcionalidades | **Banco** — query específica sem índice |
| Lentidão generalizada, banco saudável | **Infra** — rede, memória, disco |
| Queries com alta variação de tempo | **Banco** — estatísticas desatualizadas ou lock intermitente |

---

## Fase 5 — Identificação da Causa Raiz e Resolução

### Fluxo de decisão

```
Lentidão reportada após deploy
        │
        ├─► Há processos bloqueados? ──► SIM ──► Identificar bloqueador
        │         │                              └─► pg_terminate_backend (se necessário)
        │         │
        │        NÃO
        │         │
        ├─► I/O wait alto? ──► SIM ──► Verificar queries com muito Buffers: read
        │         │                    └─► Aumentar shared_buffers ou otimizar query
        │         │
        │        NÃO
        │         │
        ├─► Queries lentas no pg_stat_statements? ──► SIM ──► EXPLAIN ANALYZE
        │         │                                           └─► Índice faltante? ANALYZE?
        │         │
        │        NÃO
        │         │
        └─► Verificar camada de aplicação (N+1, connection pool, tracing)
```

### Resolução típica pós-deploy

```sql
-- Caso 1: Índice removido por migration acidentalmente
CREATE INDEX CONCURRENTLY idx_orders_customer_status
ON orders(customer_id, status)
WHERE status = 'pending';
-- CONCURRENTLY: cria o índice sem bloquear leituras/escritas

-- Caso 2: Estatísticas desatualizadas após migration com muitos dados
ANALYZE orders;

-- Caso 3: Lock de DDL travando tudo (migration que não terminou)
-- Identificar e terminar o processo da migration travada
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE query ILIKE '%ALTER TABLE%'
  AND state != 'idle'
  AND now() - query_start > interval '5 minutes';
```

---

## Evidências para Documentar a Causa Raiz

| Evidência | Como capturar |
|---|---|
| Query mais lenta identificada | Output do `pg_stat_statements` com timestamp |
| Plano de execução antes e depois da correção | `EXPLAIN ANALYZE` salvo em ambos os momentos |
| Correlação temporal deploy → lentidão | Logs do PG (`log_min_duration_statement`) cruzados com hora do deploy |
| Índice faltante confirmado | Output do `pg_indexes` comparado com homologação |
| Melhora após correção | Métricas de tempo médio antes vs depois |

> **Princípio orientador:** Diagnóstico de lentidão é eliminação de hipóteses por camada. O contexto do deploy já elimina causas aleatórias — o foco vai para o que mudou. A causa raiz raramente é complexa: na maioria dos casos pós-deploy, é um índice removido, uma query nova ineficiente ou uma transação não encerrada corretamente.
