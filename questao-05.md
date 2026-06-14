# Questão 05 — Automação e Uso de Inteligência Artificial

> **Cenário:** O papel do DBA vem evoluindo com o uso crescente de automação e Inteligência Artificial. Cite exemplos reais de como você utiliza ou utilizaria essas tecnologias para aumentar produtividade, reduzir riscos operacionais e melhorar a qualidade da administração de ambientes PostgreSQL. Quais atividades você automatizaria e quais cuidados tomaria antes de aplicar recomendações geradas por IA em produção?

---

## O DBA Moderno: de Executor a Engenheiro de Confiabilidade

O papel do DBA evoluiu de "administrador reativo que resolve problemas" para "engenheiro de confiabilidade que previne problemas e automatiza operações repetitivas". Automação e IA não substituem o DBA — eliminam o trabalho mecânico e amplificam a capacidade de análise.

A linha que separa um DBA pleno de um DBRE sênior hoje é: **o quanto do seu trabalho operacional está automatizado e quanto do seu tempo é gasto em decisões de arquitetura e melhoria contínua**.

---

## Automação: O que Automatizar e Como

### 1. Backup, Restore e Validação Automática

O maior risco de um backup é nunca ter sido testado. A solução é automatizar não apenas o backup, mas o **teste de restore**:

```bash
#!/bin/bash
# Script: validate_backup.sh
# Executa semanalmente via cron — testa o backup mais recente

set -e

BACKUP_STANZA="producao"
SANDBOX_DATA_DIR="/var/lib/postgresql/sandbox/data"
SANDBOX_PORT=5433
LOG_FILE="/var/log/pg_backup_validation_$(date +%Y%m%d).log"

echo "[$(date)] Iniciando validação de backup" >> $LOG_FILE

# 1. Restaurar backup mais recente em sandbox efêmero
rm -rf $SANDBOX_DATA_DIR
pgbackrest --stanza=$BACKUP_STANZA \
           --pg1-path=$SANDBOX_DATA_DIR \
           --type=latest \
           restore >> $LOG_FILE 2>&1

# 2. Subir instância sandbox na porta alternativa
pg_ctl -D $SANDBOX_DATA_DIR -o "-p $SANDBOX_PORT" -l $LOG_FILE start
sleep 10

# 3. Validar integridade básica
RESULTADO=$(psql -p $SANDBOX_PORT -U postgres -c "
    SELECT COUNT(*) FROM pg_database WHERE datname NOT IN ('template0','template1');
" -t -A)

if [ "$RESULTADO" -gt "0" ]; then
    echo "[$(date)] SUCESSO: Restore validado. $RESULTADO databases encontrados." >> $LOG_FILE
    # Notificar canal de monitoramento (ex: webhook Slack/Teams)
    curl -s -X POST "$WEBHOOK_URL" \
         -H 'Content-type: application/json' \
         -d "{\"text\":\"✅ Backup validado com sucesso em $(date +%Y-%m-%d)\"}"
else
    echo "[$(date)] FALHA: Restore sem databases válidos." >> $LOG_FILE
    curl -s -X POST "$WEBHOOK_URL" \
         -d "{\"text\":\"🚨 FALHA na validação de backup em $(date +%Y-%m-%d). Verificar urgente.\"}"
    exit 1
fi

# 4. Derrubar sandbox e limpar
pg_ctl -D $SANDBOX_DATA_DIR stop
rm -rf $SANDBOX_DATA_DIR

echo "[$(date)] Validação concluída." >> $LOG_FILE
```

```bash
# Agendar no cron: toda segunda-feira às 03h00
0 3 * * 1 /opt/scripts/validate_backup.sh
```

### 2. Monitoramento Inteligente de Bloat e Vacuum

```python
#!/usr/bin/env python3
# Script: autovacuum_advisor.py
# Roda a cada hora — detecta tabelas que precisam de VACUUM urgente
# e envia alerta ou executa automaticamente se abaixo de threshold seguro

import psycopg2
import requests
import os

CONN = psycopg2.connect(os.environ["PG_DSN"])
WEBHOOK = os.environ["SLACK_WEBHOOK"]
AUTO_VACUUM_THRESHOLD = 30  # % de dead tuples para alerta
AUTO_VACUUM_CRITICAL = 50   # % para execução automática (tabelas pequenas)
MAX_AUTO_SIZE_MB = 500      # limite de tamanho para vacuum automático sem aprovação

cur = CONN.cursor()
cur.execute("""
    SELECT
        schemaname,
        relname,
        n_dead_tup,
        n_live_tup,
        ROUND(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 1) AS dead_pct,
        pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS size,
        pg_total_relation_size(schemaname||'.'||relname) / 1024 / 1024 AS size_mb,
        last_autovacuum,
        last_vacuum
    FROM pg_stat_user_tables
    WHERE n_live_tup > 1000
      AND n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) > %s / 100.0
    ORDER BY dead_pct DESC
    LIMIT 20
""", (AUTO_VACUUM_THRESHOLD,))

tables = cur.fetchall()

for schema, table, dead, live, dead_pct, size, size_mb, last_auto, last_manual in tables:
    full_name = f"{schema}.{table}"

    if dead_pct >= AUTO_VACUUM_CRITICAL and size_mb <= MAX_AUTO_SIZE_MB:
        # Executar VACUUM automaticamente em tabelas pequenas com bloat crítico
        cur.execute(f"VACUUM ANALYZE {full_name}")
        CONN.commit()
        msg = f"🤖 VACUUM automático executado em `{full_name}` ({dead_pct}% dead tuples, {size})"
    else:
        # Apenas alertar para tabelas grandes ou com bloat moderado
        msg = f"⚠️ Bloat em `{full_name}`: {dead_pct}% dead tuples ({dead:,} linhas mortas, {size})"

    requests.post(WEBHOOK, json={"text": msg})

cur.close()
CONN.close()
```

### 3. Infrastructure as Code (IaC) para Provisionamento

```hcl
# terraform/rds_postgres.tf
# Provisionar instância PostgreSQL com configurações padronizadas

resource "aws_db_instance" "producao" {
  identifier        = "producao-postgres-16"
  engine            = "postgres"
  engine_version    = "16.2"
  instance_class    = "db.r6g.xlarge"
  allocated_storage = 500
  storage_type      = "gp3"
  storage_encrypted = true

  db_name  = "myapp"
  username = "admin"
  password = var.db_password  # gerenciado pelo Vault, nunca hardcoded

  # Parâmetros de performance configurados via parameter group
  parameter_group_name = aws_db_parameter_group.pg16_optimized.name

  # Backup automatizado
  backup_retention_period = 14
  backup_window           = "03:00-04:00"
  maintenance_window      = "Mon:04:00-Mon:05:00"

  # Multi-AZ para alta disponibilidade
  multi_az               = true
  deletion_protection    = true

  # Monitoramento
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_enhanced_monitoring.arn

  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
    Owner       = "dba-team"
  }
}

resource "aws_db_parameter_group" "pg16_optimized" {
  name   = "pg16-production-optimized"
  family = "postgres16"

  parameter {
    name  = "shared_buffers"
    value = "{DBInstanceClassMemory/4}"  # 25% da RAM
  }
  parameter {
    name  = "effective_cache_size"
    value = "{DBInstanceClassMemory*3/4}"  # 75% da RAM
  }
  parameter {
    name  = "log_min_duration_statement"
    value = "2000"  # loga queries acima de 2 segundos
  }
  parameter {
    name  = "pgaudit.log"
    value = "ddl,role"
  }
}
```

### 4. Automação de Alertas com Lógica de Contexto

```yaml
# prometheus/alerts/postgresql.yml
# Regras de alerta inteligentes — consideram contexto antes de disparar

groups:
  - name: postgresql_critical
    rules:
      - alert: BackupNaoExecutado
        expr: time() - pg_backup_last_success_timestamp > 86400  # 24 horas
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Backup não executado há mais de 24h"
          description: "Último backup bem-sucedido foi às {{ $value | humanizeTimestamp }}"

      - alert: ReplicacaoAtrasada
        expr: pg_replication_lag_seconds > 30
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Lag de replicação acima de 30 segundos"

      - alert: ConexoesEsgotando
        expr: pg_stat_activity_count / pg_settings_max_connections > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Conexões acima de 85% do limite ({{ $value | humanizePercentage }})"
```

---

## Inteligência Artificial: Como Uso no Dia a Dia

### 1. Interpretação de EXPLAIN ANALYZE

Planos de execução complexos com múltiplos joins, CTEs e subqueries podem ter 200+ linhas. Uso IA como **primeiro filtro de análise**:

```
Prompt que funciona bem:
"Analise este plano de execução PostgreSQL e identifique:
1. Os nós mais custosos e por quê
2. Estimativas de rows muito divergentes da realidade (indica statistics problem)
3. Algoritmos de join que podem ser melhorados
4. Possibilidade de índices faltantes

[colar output do EXPLAIN ANALYZE aqui]"
```

**O que faço com a análise:** trato como hipótese a ser validada, não como diagnóstico definitivo. A IA pode sugerir um índice — eu verifico se o índice já existe, se a cardinalidade justifica, e testo em staging antes de criar em produção.

### 2. Geração e Otimização de Queries

Para queries analíticas complexas ou SQL que envolve features menos comuns (window functions, CTEs recursivas, LATERAL joins), uso IA para gerar o rascunho inicial:

```
Prompt:
"Escreva uma query PostgreSQL que calcule a mediana de tempo entre pedidos
consecutivos por cliente, considerando apenas os últimos 90 dias e clientes
com pelo menos 3 pedidos. Use window functions."
```

**Processo após geração:** revisar a lógica, testar em staging com `EXPLAIN ANALYZE`, validar resultado com amostra de dados conhecida, e só então aplicar.

### 3. Ferramentas com IA Integrada para PostgreSQL

| Ferramenta | O que faz com IA |
|---|---|
| **pganalyze** | Analisa `pg_stat_statements` e sugere índices automaticamente, detecta regressões de queries |
| **Datadog APM** | Correlaciona lentidão de banco com deploys, detecta anomalias de performance |
| **OtterTune** | Ajuste automático de parâmetros do PostgreSQL baseado em ML (workload-specific) |
| **GitHub Copilot** | Geração de scripts de automação, queries SQL, migrations |

### 4. Documentação e Runbooks Automatizados

```
Uso IA para:
- Gerar documentação de schemas a partir do DDL das tabelas
- Criar runbooks de procedimentos operacionais
- Redigir post-mortems de incidentes baseado em timeline de eventos
- Transformar planos de execução em explicações em linguagem natural
  (útil para explicar problemas de banco para times de desenvolvimento)
```

---

## Cuidados Antes de Aplicar Recomendações de IA em Produção

### O que NUNCA fazer

| Ação | Risco |
|---|---|
| Rodar query gerada por IA diretamente em produção | Query pode ter efeitos colaterais não previstos |
| Enviar dados reais de clientes para LLMs públicos | Violação de LGPD, privacidade e compliance |
| Aplicar ajuste de parâmetro sugerido por IA sem entender o impacto | Configurações de PG têm efeitos em cascata (ex: `work_mem` alto × conexões simultâneas) |
| Confiar em documentação gerada por IA sem verificar versão | A IA pode referenciar comportamentos de versões antigas do PostgreSQL |

### Processo de Validação em 3 Etapas

```
Recomendação da IA
       │
  1. ENTENDER
  │  └─► "Por que isso funcionaria? Qual o mecanismo?"
  │      Se não consigo explicar, não aplico.
       │
  2. TESTAR em staging
  │  └─► EXPLAIN ANALYZE, verificação de resultado, teste de carga
  │      Comparação com baseline antes da mudança
       │
  3. MONITORAR após aplicar em produção
     └─► Métricas por 24-48h
         Alerta configurado para regressão
         Rollback planejado
```

### Exemplo Concreto: Ajuste de `work_mem`

```
Cenário: IA sugere aumentar work_mem de 4MB para 64MB para resolver sort em disco.

Antes de aplicar:
1. Calcular impacto: work_mem × max_connections × 3 (sorts paralelos por query)
   64MB × 200 conexões × 3 = 38.4 GB de memória no pior caso
   Se o servidor tem 32GB de RAM → pode causar OOM killer

2. Verificar alternativa: criar índice na coluna de sort (resolve sem risco)

3. Se realmente necessário: aplicar apenas para a role específica que precisa
   ALTER ROLE relatorio_bi SET work_mem = '64MB';
   (não para todas as conexões)

4. Monitorar com: SELECT * FROM pg_stat_activity WHERE wait_event = 'BufferPin';
```

---

## O que Não Automatizaria (e Por Que)

| Atividade | Por que manter controle humano |
|---|---|
| `VACUUM FULL` em tabelas grandes | Bloqueia completamente — exige janela e comunicação |
| `pg_terminate_backend` automático em locks | Pode derrubar transação crítica de negócio |
| Aplicação automática de parâmetros de PG | Configurações erradas causam crash do servidor |
| Failover automático sem validação | Risco de split-brain em redes particionadas |
| Execução de migrations em produção sem revisão | Alterações de schema são irreversíveis em muitos casos |

> **Princípio orientador:** Automatizo tudo que é repetitivo, previsível e baixo risco. Mantenho controle humano em tudo que é irreversível, tem impacto amplo ou exige julgamento de contexto que a automação não tem. IA é um multiplicador do meu trabalho — ela acelera minha análise e geração de código, mas a responsabilidade e o julgamento final são sempre meus.

---

## Resumo: Automação vs Controle Humano

```
AUTOMATIZO COMPLETAMENTE          SUPERVISÃO HUMANA OBRIGATÓRIA
─────────────────────────         ──────────────────────────────
Backup + validação de restore     VACUUM FULL em produção
Alertas de monitoramento          Failover de instância primária
ANALYZE em tabelas pequenas       Migrations de schema crítico
Provisionamento via Terraform     Ajustes de parâmetros de PG
Rotação de credenciais            Terminate em processos ativos
Relatórios de auditoria           Qualquer DDL em produção
Deploy de migrations aprovadas    Decisões de arquitetura
```
