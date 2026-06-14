# Questão 02 — Atualização sem Impacto ao Negócio

> **Cenário:** A empresa possui um PostgreSQL em versão próxima ao fim do suporte e precisa realizar uma atualização para uma versão mais recente. O ambiente atende milhares de usuários e possui restrições de indisponibilidade. Descreva como você planejaria essa atualização, quais riscos avaliaria previamente e quais estratégias utilizaria para garantir uma transição segura e com o menor impacto possível.

---

## Premissa: Upgrade Major é um Projeto, Não uma Tarefa

Uma atualização de versão major do PostgreSQL (ex: 13 → 16) envolve mudanças internas de formato de dados, comportamento de queries e compatibilidade de extensões. Em um ambiente com milhares de usuários e restrições de indisponibilidade, isso exige **planejamento estruturado, execução em fases e plano de rollback claro**.

---

## Fase 1 — Avaliação de Riscos (2 a 4 semanas antes)

### 1.1 Diagnóstico de Compatibilidade com `pg_upgrade --check`

O primeiro passo é executar o `pg_upgrade` em modo de verificação, **sem realizar nenhuma alteração**, apenas para identificar incompatibilidades:

```bash
# Instalar a nova versão do PostgreSQL (ex: 16) em paralelo no servidor
# sem remover a versão atual (ex: 13)

pg_upgrade \
  --old-datadir /var/lib/postgresql/13/main \
  --new-datadir /var/lib/postgresql/16/main \
  --old-bindir /usr/lib/postgresql/13/bin \
  --new-bindir /usr/lib/postgresql/16/bin \
  --check  # <-- modo diagnóstico, sem alterações
```

O relatório gerado aponta:
- Tipos de dados incompatíveis (ex: `unknown` type foi removido em versões recentes)
- Encoding incompatível entre clusters
- Extensões que precisam ser recompiladas ou atualizadas
- Funções ou sintaxes SQL depreciadas

### 1.2 Inventário de Extensões

```sql
-- Listar todas as extensões instaladas e suas versões
SELECT name, default_version, installed_version, comment
FROM pg_available_extensions
WHERE installed_version IS NOT NULL
ORDER BY name;
```

Para cada extensão, verificar:
- Suporte na versão alvo (ex: PostGIS, pgcrypto, pg_stat_statements, pgAudit)
- Se há breaking changes no upgrade da extensão em si
- Se há versão disponível nos repositórios para a nova versão do PG

### 1.3 Análise de Queries e Features Depreciadas

```sql
-- Ativar log de queries depreciadas (em homologação)
ALTER SYSTEM SET log_min_messages = 'WARNING';
SELECT pg_reload_conf();

-- Verificar uso de operadores/funções removidos na versão alvo
-- (ex: operadores de geometria no PostGIS, funções de to_tsquery)
```

Complementar com análise do `pg_stat_statements` para identificar as queries mais executadas e validá-las na versão alvo:

```sql
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 50;
```

### 1.4 Avaliação de Riscos Mapeados

| Risco | Impacto | Mitigação |
|---|---|---|
| Extensão incompatível | Alto — pode impedir upgrade | Atualizar extensão antes ou encontrar alternativa |
| Query com sintaxe depreciada | Médio — erro em produção pós-upgrade | Corrigir na aplicação antes do upgrade |
| Regressão de performance | Alto — usuários afetados sem falha explícita | Testes de carga com EXPLAIN em ambiente idêntico |
| Falha durante upgrade | Crítico — downtime estendido | Plano de rollback detalhado e testado |
| Incompatibilidade de driver | Médio | Validar versão do driver JDBC/psycopg na aplicação |

---

## Fase 2 — Ambiente de Homologação Idêntico à Produção

Antes de qualquer execução em produção, o upgrade é realizado integralmente em um ambiente de homologação com:
- **Mesmo hardware** ou perfil de recursos equivalente
- **Cópia real dos dados** de produção (restore do backup mais recente)
- **Mesmas extensões** e configurações de `postgresql.conf`

```bash
# Restaurar backup de produção em homologação
pgbackrest --stanza=producao --type=latest restore \
  --pg1-path=/var/lib/postgresql/13/main_hml

# Executar upgrade completo em homologação
pg_upgrade \
  --old-datadir /var/lib/postgresql/13/main_hml \
  --new-datadir /var/lib/postgresql/16/main_hml \
  --old-bindir /usr/lib/postgresql/13/bin \
  --new-bindir /usr/lib/postgresql/16/bin \
  --link  # hard links: mais rápido, sem duplicar dados em disco
```

### Testes de validação em homologação

1. Executar as 50 queries mais críticas do `pg_stat_statements` e comparar planos de execução
2. Executar suite de testes de integração da aplicação
3. Simular carga equivalente a produção (ex: `pgbench` com workload customizado)
4. Medir e comparar tempos de resposta antes e depois

---

## Fase 3 — Estratégia de Upgrade com Mínimo Downtime

Para um ambiente com restrições severas de indisponibilidade, a estratégia recomendada é **upgrade via Replicação Lógica**, que permite migrar os dados de forma contínua enquanto a versão antiga continua em operação.

### Estratégia A: Logical Replication (downtime de segundos a poucos minutos)

```
[PG 13 - Produção Atual]  ──── Logical Replication ────►  [PG 16 - Nova Instância]
         ↑                                                           ↑
   Recebe tráfego                                         Sincronizando dados
         │                                                           │
         └────────── Chaveamento (failover DNS/LB) ────────────────►│
                         (janela de ~30 segundos)              Passa a receber tráfego
```

**Passo a passo:**

```sql
-- No PG 13 (origem): habilitar publicação lógica
-- Requer wal_level = logical (pode exigir restart se ainda não configurado)
ALTER SYSTEM SET wal_level = 'logical';
-- (restart do PG 13 se necessário — planejar janela para isso)

-- Criar publicação de todas as tabelas
CREATE PUBLICATION upgrade_pub FOR ALL TABLES;
```

```sql
-- No PG 16 (destino): criar estrutura e subscription
-- 1. Aplicar o schema (DDL) do banco atual no PG 16
-- pg_dump --schema-only e restaurar no PG 16

-- 2. Criar subscription apontando para o PG 13
CREATE SUBSCRIPTION upgrade_sub
  CONNECTION 'host=pg13-host port=5432 dbname=mydb user=replicator password=...'
  PUBLICATION upgrade_pub;
```

```sql
-- Monitorar sincronização (aguardar até sync_state = 'streaming')
SELECT subname, pid, received_lsn, latest_end_lsn, 
       (latest_end_lsn - received_lsn) AS lag_bytes
FROM pg_stat_subscription;
```

**Chaveamento final (janela mínima):**
```sql
-- 1. Bloquear novas escritas no PG 13 (aplicação ou proxy)
-- 2. Aguardar lag_bytes = 0 (replicação totalmente sincronizada)
-- 3. Dropar a subscription no PG 16
DROP SUBSCRIPTION upgrade_sub;
-- 4. Atualizar DNS/load balancer para apontar para PG 16
-- 5. Validar funcionamento e liberar tráfego
```

### Estratégia B: pg_upgrade com --link + Failover de Réplica (downtime de minutos)

Quando Logical Replication não é viável (ex: `wal_level` não está como `logical` e não há janela para mudá-lo):

1. Fazer upgrade da réplica standby para PG 16 usando `pg_upgrade`
2. Promover a réplica como novo primary
3. O primary antigo (PG 13) vira o ponto de rollback

```
[PG 13 Primary] ──streaming replication──► [PG 13 Standby]
                                                  │
                                          pg_upgrade --link
                                                  │
                                            [PG 16 Standby]
                                                  │
                                     pg_ctl promote (failover)
                                                  │
                                            [PG 16 Primary] ← tráfego
```

**Vantagem:** Downtime reduzido ao tempo de failover (~1-2 minutos)
**Desvantagem:** Não há caminho de volta automático — rollback exige re-sincronização

---

## Fase 4 — Plano de Rollback

O plano de rollback deve estar **documentado e testado** antes da execução em produção.

| Estratégia usada | Rollback disponível | Tempo de rollback |
|---|---|---|
| Logical Replication | Redirecionar tráfego de volta ao PG 13 (ainda operacional) | Segundos |
| pg_upgrade --link | Restaurar backup pré-upgrade | Minutos a horas |
| pg_upgrade sem --link | Iniciar instância PG 13 original (arquivos intactos) | Minutos |

**Critérios de ativação do rollback:**
- Erro crítico nos primeiros 15 minutos pós-upgrade
- Taxa de erro da aplicação acima do threshold definido
- Query crítica com regressão de performance > 3x

---

## Fase 5 — Pós-Upgrade: Validação e Otimização

```sql
-- Atualizar estatísticas (obrigatório após pg_upgrade)
ANALYZE;

-- Recriar índices (pg_upgrade não garante que estão otimizados para nova versão)
REINDEX DATABASE nome_do_banco;

-- Atualizar extensões para versão compatível com PG 16
ALTER EXTENSION pg_stat_statements UPDATE;
ALTER EXTENSION pgcrypto UPDATE;
-- (repetir para todas as extensões instaladas)

-- Verificar se há objetos inválidos
SELECT schemaname, viewname
FROM pg_views
WHERE definition ILIKE '%deprecated_function%';
```

```bash
# Validar logs nas primeiras horas pós-upgrade
tail -f /var/log/postgresql/postgresql-16-main.log | grep -E "ERROR|FATAL|WARNING"
```

---

## Resumo da Estratégia de Decisão

```
Qual é a tolerância a downtime?

├── Zero (segundos)
│     └── Logical Replication (PG 10+, requer wal_level = logical)

├── Baixa (1-5 minutos)
│     └── pg_upgrade na réplica + failover controlado

└── Moderada (30-60 minutos, janela noturna)
      └── pg_upgrade --link direto no primary com backup validado
```

> **Princípio orientador:** O upgrade em si é a parte mais simples. O trabalho real está na preparação — compatibilidade, testes em homologação idêntica à produção e rollback planejado. Um upgrade sem esses passos é um risco desnecessário, independentemente do tamanho do ambiente.
