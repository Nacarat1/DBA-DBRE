# Questão 01 — Assumindo um Ambiente Crítico

> **Cenário:** Você assume a responsabilidade por um ambiente PostgreSQL de produção que suporta sistemas críticos da empresa. Não existe documentação atualizada, não há histórico recente de testes de restauração e ninguém consegue garantir que os backups estejam funcionando corretamente. Quais seriam suas prioridades nas primeiras 72 horas e quais evidências você buscaria para considerar o ambiente minimamente seguro?

---

## Premissa Fundamental

Antes de qualquer diagnóstico de performance, tuning ou documentação, há uma realidade inegociável: **um ambiente sem backup validado é um ambiente em risco de perda total de dados**. A ausência de garantia sobre os backups é tratada como incidente ativo, não como pendência de backlog.

A primeira ação não é investigar — é **proteger**.

---

## Fase 1 — Primeiras 2 Horas: Backup Imediato

### 1.1 Criar um backup físico consistente antes de qualquer intervenção

Independentemente de qualquer backup existente, a primeira ação é gerar um novo backup físico de base com estado conhecido e controlado.

```bash
# Verificar conectividade e versão do PostgreSQL
psql -U postgres -c "SELECT version();"

# Verificar espaço disponível no servidor (e no destino do backup)
df -h

# Executar backup físico completo com pg_basebackup
# -Ft: formato tar | -z: compressão gzip | -P: progresso | -R: prepara recovery.conf
pg_basebackup \
  -h localhost \
  -U replicator \
  -D /backup/basebackup_$(date +%Y%m%d_%H%M%S) \
  -Ft -z -P -R \
  --wal-method=stream \
  --checkpoint=fast

# Verificar integridade do arquivo gerado
ls -lh /backup/basebackup_*/
```

> **Por que `--wal-method=stream`?** Garante que os WALs gerados *durante* o backup também sejam capturados, evitando janelas de inconsistência. O `--checkpoint=fast` reduz o tempo de início do backup.

### 1.2 Confirmar que WALs estão sendo arquivados (se archive_mode = on)

```sql
-- Verificar configuração de arquivamento de WAL
SHOW archive_mode;
SHOW archive_command;
SHOW archive_status;

-- Verificar último WAL arquivado com sucesso
SELECT * FROM pg_stat_archiver;
```

Se `last_failed_time` for mais recente que `last_archived_time`, o arquivamento de WALs está falhando — isso elimina a possibilidade de PITR e é um risco crítico adicional.

---

## Fase 2 — Primeiras 4 Horas: Validação por Restore em Sandbox

Um backup que nunca foi restaurado **não é um backup, é uma esperança**.

### 2.1 Provisionar ambiente sandbox isolado

```bash
# Em servidor separado (ou container Docker isolado):
docker run -d \
  --name pg_restore_test \
  -e POSTGRES_PASSWORD=test \
  -v /backup/basebackup_YYYYMMDD_HHMMSS:/backup \
  postgres:15

# Dentro do container, restaurar o backup
docker exec -it pg_restore_test bash

# Descompactar e restaurar
mkdir -p /var/lib/postgresql/data_restored
tar -xzf /backup/base.tar.gz -C /var/lib/postgresql/data_restored/
tar -xzf /backup/pg_wal.tar.gz -C /var/lib/postgresql/data_restored/pg_wal/

# Iniciar a instância restaurada (PostgreSQL 12+)
# Criar standby.signal para confirmar que é um restore
touch /var/lib/postgresql/data_restored/standby.signal
```

### 2.2 Validação funcional pós-restore

```sql
-- Conectar na instância restaurada e verificar consistência
SELECT pg_is_in_recovery(); -- deve retornar false após recovery completo

-- Verificar databases existentes
SELECT datname, pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;

-- Verificar integridade das tabelas mais críticas
-- (adaptar conforme o schema real identificado)
SELECT schemaname, tablename, n_live_tup, n_dead_tup, last_vacuum, last_analyze
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC
LIMIT 20;

-- Contar registros em tabelas-chave (comparar com produção)
SELECT COUNT(*) FROM schema_critico.tabela_principal;
```

**Evidência gerada:** Restore bem-sucedido com timestamp, contagem de registros consistente e instância operacional. Este resultado deve ser documentado formalmente.

---

## Fase 3 — Primeiras 24 Horas: Levantamento do Ambiente

Com os dados protegidos, inicia-se o diagnóstico sem pressão de risco imediato.

### 3.1 Topologia e Configuração Geral

```sql
-- Versão exata
SELECT version();

-- É primary ou standby?
SELECT pg_is_in_recovery(), inet_server_addr(), current_database();

-- Identificar réplicas conectadas (se for primary)
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       (sent_lsn - replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;

-- Configurações críticas
SELECT name, setting, unit, context
FROM pg_settings
WHERE name IN (
  'max_connections', 'shared_buffers', 'work_mem', 'maintenance_work_mem',
  'effective_cache_size', 'wal_level', 'max_wal_senders',
  'checkpoint_completion_target', 'log_min_duration_statement',
  'autovacuum', 'archive_mode', 'archive_command'
);
```

### 3.2 Saúde Geral das Tabelas

```sql
-- Identificar tabelas com alto dead tuple ratio (candidatas a bloat/vacuum urgente)
SELECT schemaname, tablename,
       n_dead_tup,
       n_live_tup,
       ROUND(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_ratio_pct,
       last_vacuum,
       last_autovacuum,
       last_analyze
FROM pg_stat_user_tables
WHERE n_live_tup > 1000
ORDER BY dead_ratio_pct DESC
LIMIT 20;

-- Verificar tamanho de tabelas e índices
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
       pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
       pg_size_pretty(pg_indexes_size(schemaname||'.'||tablename)) AS index_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;
```

### 3.3 Atividade e Locks

```sql
-- Conexões ativas e queries em execução
SELECT pid, usename, application_name, client_addr, state,
       now() - query_start AS duration,
       LEFT(query, 100) AS query_preview
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- Locks bloqueantes
SELECT blocking_locks.pid AS blocking_pid,
       blocked_locks.pid AS blocked_pid,
       blocked_activity.usename AS blocked_user,
       blocking_activity.query AS blocking_query,
       blocked_activity.query AS blocked_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
  AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
  AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

### 3.4 Uso de Disco e Crescimento

```bash
# Uso de disco por diretório do PostgreSQL
du -sh $PGDATA/*
du -sh $PGDATA/pg_wal/

# Projetar crescimento: tamanho dos WALs gerados na última hora
ls -lt $PGDATA/pg_wal/ | head -20
```

---

## Fase 4 — 24h a 48h: Estruturar Rotina de Backup e Alertas

### 4.1 Implementar solução de backup gerenciada (se não houver pgBackRest/Barman)

```ini
# Exemplo de configuração mínima com pgBackRest (/etc/pgbackrest/pgbackrest.conf)
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
log-level-console=info
log-level-file=detail

[producao]
pg1-path=/var/lib/postgresql/15/main

# Executar primeiro backup completo
pgbackrest --stanza=producao --log-level-console=info stanza-create
pgbackrest --stanza=producao --type=full backup

# Verificar integridade do backup
pgbackrest --stanza=producao info
```

### 4.2 Agendar backup e teste de restore automático

```bash
# Crontab: backup incremental diário às 01h, full aos domingos às 00h
0 1 * * 1-6 pgbackrest --stanza=producao --type=incr backup
0 0 * * 0   pgbackrest --stanza=producao --type=full backup

# Teste de restore automatizado semanal (script simplificado)
# Executar restore em ambiente efêmero e registrar resultado
```

### 4.3 Alertas mínimos indispensáveis

| Métrica | Threshold | Ação |
|---|---|---|
| Espaço em disco (PGDATA) | > 80% | Alerta CRÍTICO |
| Lag de replicação | > 30s | Alerta ALTO |
| Último backup bem-sucedido | > 25h | Alerta CRÍTICO |
| Arquivamento de WAL falhou | Qualquer falha | Alerta CRÍTICO |
| Long-running queries | > 5 min | Alerta ALTO |
| Conexões ativas | > 80% de max_connections | Alerta MÉDIO |

---

## Fase 5 — 48h a 72h: Documentação Mínima e Verificação de Acessos

### 5.1 Inventário de usuários e privilégios

```sql
-- Listar roles e seus atributos
SELECT rolname, rolsuper, rolcreatedb, rolcreaterole, rolcanlogin, rolconnlimit
FROM pg_roles
ORDER BY rolsuper DESC, rolname;

-- Listar memberships
SELECT r.rolname AS role, m.rolname AS member
FROM pg_auth_members am
JOIN pg_roles r ON r.oid = am.roleid
JOIN pg_roles m ON m.oid = am.member;

-- Verificar quem tem acesso superuser (deve ser mínimo ou nenhum além do postgres)
SELECT rolname FROM pg_roles WHERE rolsuper = true;
```

### 5.2 Cheklist de Documentação Mínima

- [ ] Versão do PostgreSQL, SO e configuração de hardware
- [ ] Topologia: Primary / Réplicas / Connection Pooler (PgBouncer?)
- [ ] Ferramenta de backup utilizada e localização dos arquivos
- [ ] Jobs recorrentes (pg_cron, crontab, scripts externos)
- [ ] Contatos de escalonamento de incidentes
- [ ] Janelas de manutenção acordadas com o negócio

---

## Evidências para Considerar o Ambiente Minimamente Seguro

| # | Evidência | Como Verificar |
|---|---|---|
| 1 | **Backup full executado e validado** | Log de pg_basebackup/pgBackRest + restore de sucesso documentado |
| 2 | **Teste de restore bem-sucedido em sandbox** | Script de validação com comparação de row counts |
| 3 | **WAL archiving funcionando** | `SELECT * FROM pg_stat_archiver` sem falhas recentes |
| 4 | **Replicação síncrona ou assíncrona ativa e saudável** | `pg_stat_replication` com lag < threshold definido |
| 5 | **Alertas de monitoramento ativos** | Dashboard/alertas configurados e testados |
| 6 | **Sem superusers desnecessários** | Inventário de roles auditado |
| 7 | **Sem tabelas em risco crítico de bloat** | dead_ratio_pct < 20% nas tabelas principais |
| 8 | **Documentação mínima registrada** | Documento de arquitetura versionado |

---

## Resumo Executivo das 72 Horas

```
Hora  0-02h → Criar backup físico imediato (pg_basebackup com WAL streaming)
Hora  02-04h → Validar backup via restore completo em ambiente isolado
Hora  04-24h → Mapear topologia, configurações críticas, saúde de tabelas e acessos
Hora  24-48h → Implementar solução de backup gerenciada + agendamento + alertas
Hora  48-72h → Documentação mínima, auditoria de acessos, comunicação ao time
```

> **Princípio orientador:** Em um ambiente sem backup garantido, cada hora sem um backup validado é uma hora de exposição ao risco de perda total de dados. Todo o restante — performance, tuning, documentação — é secundário enquanto esse risco não for eliminado.
