# Questão 04 — Segurança e Governança de Banco de Dados

> **Cenário:** Imagine que você precisa implantar controles em um ambiente onde desenvolvedores possuem acesso direto ao banco de produção e alterações são realizadas sem processo formal de aprovação. Como você estruturaria a governança de mudanças, o controle de acessos e a rastreabilidade das operações para aumentar a segurança sem comprometer a agilidade das equipes?

---

## Premissa: Segurança sem Processo é Bloqueio. Processo sem Segurança é Risco.

O desafio não é apenas técnico — é cultural. Um ambiente onde desenvolvedores acessam produção diretamente geralmente existe porque, em algum momento, foi a solução mais rápida para resolver um problema. Propor controles que tornam o trabalho do dia a dia mais lento do que antes gera resistência e boicote passivo.

A abordagem correta é: **remover o acesso direto e substituir por processos que são igualmente ágeis ou mais rápidos do que o acesso manual atual**.

---

## Pilar 1 — Controle de Acessos (Princípio do Menor Privilégio)

### 1.1 Auditoria do Estado Atual

```sql
-- Inventário de todos os usuários e seus privilégios
SELECT
    r.rolname,
    r.rolsuper,
    r.rolcreatedb,
    r.rolcreaterole,
    r.rolcanlogin,
    r.rolconnlimit,
    ARRAY(
        SELECT b.rolname FROM pg_catalog.pg_auth_members m
        JOIN pg_catalog.pg_roles b ON m.roleid = b.oid
        WHERE m.member = r.oid
    ) AS member_of
FROM pg_catalog.pg_roles r
ORDER BY r.rolsuper DESC, r.rolcanlogin DESC, r.rolname;

-- Quem tem acesso a quais schemas/tabelas?
SELECT grantee, table_schema, table_name, privilege_type
FROM information_schema.role_table_grants
WHERE grantee NOT IN ('postgres', 'PUBLIC')
ORDER BY grantee, table_schema, table_name;
```

### 1.2 Estrutura de Roles por Responsabilidade

```sql
-- Criar roles funcionais (nunca dar acesso diretamente a usuários individuais)

-- Role para aplicação em produção (apenas o necessário)
CREATE ROLE app_producao NOLOGIN;
GRANT CONNECT ON DATABASE mydb TO app_producao;
GRANT USAGE ON SCHEMA public TO app_producao;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_producao;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_producao;

-- Role somente leitura (para analistas, suporte, BI)
CREATE ROLE readonly_producao NOLOGIN;
GRANT CONNECT ON DATABASE mydb TO readonly_producao;
GRANT USAGE ON SCHEMA public TO readonly_producao;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_producao;

-- Role para DBA (acesso restrito a operações administrativas)
CREATE ROLE dba_operacoes NOLOGIN;
GRANT readonly_producao TO dba_operacoes;
GRANT CREATE ON SCHEMA public TO dba_operacoes; -- apenas DDL controlado

-- Usuários individuais herdam roles, não recebem acesso direto
CREATE USER joao_silva WITH PASSWORD '...' LOGIN;
GRANT readonly_producao TO joao_silva;

-- Garantir que roles futuras também herdem as permissões corretamente
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO readonly_producao;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_producao;
```

### 1.3 Remover Superusers Desnecessários

```sql
-- Rebaixar usuários com superuser desnecessário
ALTER ROLE desenvolvedor_fulano NOSUPERUSER;

-- Revogar acessos diretos de escrita em produção de desenvolvedores
REVOKE INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public FROM desenvolvedor_fulano;

-- O desenvolvedor passa a ter apenas leitura em produção (para debug)
GRANT readonly_producao TO desenvolvedor_fulano;
```

### 1.4 Acesso via Bastion / Vault (eliminar credenciais fixas)

Em vez de credenciais fixas no código ou no cliente de banco:

```
Desenvolvedor
     │
     ▼
[Bastion Host / Teleport / HashiCorp Vault]
     │
     ├─► Autenticação com MFA + SSO corporativo
     │
     ▼
[PostgreSQL Produção]
     │
     └─► Credencial temporária gerada dinamicamente (TTL: 1 hora)
         Revogada automaticamente após expiração
```

Benefícios:
- Sem credenciais hardcoded em `~/.pgpass` de desenvolvedores
- Audit trail completo: quem acessou, quando e por quanto tempo
- Credencial expira automaticamente — sem "senha do DBA que nunca muda"

### 1.5 Mascaramento de Dados para Ambientes Não-Produtivos

```sql
-- Desenvolvedores não precisam de dados reais — usar dados anonimizados em dev/staging
-- Extensão postgresql_anonymizer
SELECT anon.init();

-- Definir regras de mascaramento por coluna
SECURITY LABEL FOR anon ON COLUMN customers.cpf
    IS 'MASKED WITH FUNCTION anon.partial(cpf, 3, ''*****'', 3)';

SECURITY LABEL FOR anon ON COLUMN customers.email
    IS 'MASKED WITH FUNCTION anon.fake_email()';

-- Exportar dump anonimizado para staging
pg_dump --no-security-labels mydb | psql staging_db
```

---

## Pilar 2 — Governança de Mudanças (Schema Change Management)

### 2.1 Todas as Alterações de Schema via Migrations Versionadas

Nenhuma alteração direta em produção. Todo DDL passa por:

```
Desenvolvedor
     │ cria migration file
     ▼
[Git Repository]
     │ Pull Request
     ▼
[Code Review] ← DBA revisa migrations críticas
     │ Aprovação
     ▼
[CI/CD Pipeline]
     │ roda Flyway/Liquibase em staging
     │ testa e valida
     ▼
[Deploy em Produção]
     │ migration aplicada automaticamente
     ▼
[Histórico versionado e rastreável]
```

```sql
-- Exemplo de migration Flyway (V20240613__add_index_orders_customer.sql)
-- Convenção: V<versão>__<descrição>.sql

-- Criar índice sem bloquear produção
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_customer_id
ON public.orders(customer_id);

-- Registrar motivo no comentário (rastreabilidade)
COMMENT ON INDEX idx_orders_customer_id IS
    'Criado em 2024-06-13 para melhorar performance da listagem de pedidos por cliente. PR #342';
```

### 2.2 Regras de Aprovação por Risco da Alteração

| Tipo de Alteração | Risco | Processo |
|---|---|---|
| `CREATE INDEX CONCURRENTLY` | Baixo | 1 revisor (DBA) |
| `ALTER TABLE ADD COLUMN DEFAULT NULL` | Baixo | 1 revisor |
| `ALTER TABLE ADD COLUMN NOT NULL` | Médio | DBA + Tech Lead |
| `DROP TABLE` / `DROP COLUMN` | Alto | DBA + Tech Lead + Aprovação manual |
| `UPDATE` em massa sem WHERE | Crítico | Proibido via pipeline — exige janela de manutenção |

### 2.3 Proteção contra Alterações Manuais em Produção

```sql
-- Revogar permissão de DDL diretamente para usuários de aplicação e desenvolvedores
REVOKE CREATE ON SCHEMA public FROM desenvolvedor_fulano;
REVOKE CREATE ON DATABASE mydb FROM desenvolvedor_fulano;

-- Apenas a role do pipeline de CI/CD tem permissão de DDL
CREATE ROLE cicd_migrations NOLOGIN;
GRANT CREATE ON SCHEMA public TO cicd_migrations;
GRANT ALL ON DATABASE mydb TO cicd_migrations;

-- Usuário específico do pipeline (com senha rotacionada automaticamente)
CREATE USER pipeline_user LOGIN;
GRANT cicd_migrations TO pipeline_user;
```

---

## Pilar 3 — Rastreabilidade e Auditoria

### 3.1 Habilitar pgAudit

A extensão `pgAudit` registra em log DDL e DML específicos com contexto completo (usuário, timestamp, objeto afetado, query completa).

```ini
# postgresql.conf
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'ddl, role, connection'  # o que auditar
pgaudit.log_catalog = off              # evita poluição com queries do sistema
pgaudit.log_relation = on              # inclui o objeto afetado
pgaudit.log_statement_once = off       # loga cada statement individualmente
```

```sql
-- Após configurar e reiniciar o PostgreSQL
CREATE EXTENSION pgaudit;

-- Auditoria granular por role (ex: auditar tudo que o pipeline faz)
ALTER ROLE pipeline_user SET pgaudit.log = 'all';

-- Exemplo do que o pgAudit registra:
-- AUDIT: SESSION,1,1,DDL,CREATE INDEX,,,CREATE INDEX CONCURRENTLY idx_orders...
-- AUDIT: SESSION,2,1,ROLE,GRANT,,, GRANT SELECT ON TABLE orders TO readonly_producao
```

### 3.2 Log de Queries Lentas e DDL no postgresql.conf

```ini
# postgresql.conf — configurações de log
log_destination = 'csvlog'
logging_collector = on
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-%Y-%m-%d.log'
log_rotation_age = 1d
log_rotation_size = 100MB

# Loga DDL sempre
log_statement = 'ddl'

# Loga queries lentas (acima de 2 segundos)
log_min_duration_statement = 2000

# Inclui usuário, database e IP em cada linha de log
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '
```

### 3.3 Centralização de Logs (SIEM)

Os logs do PostgreSQL devem ser enviados para uma ferramenta centralizada:

```
PostgreSQL Logs
     │
     ▼
[Filebeat / Fluentd / Vector]   ← agente de coleta
     │
     ▼
[Elasticsearch / Splunk / Datadog Logs]   ← armazenamento
     │
     ▼
[Dashboard + Alertas]
     │
     ├─► Alerta: DDL executado fora do pipeline (possível acesso manual)
     ├─► Alerta: Acesso de usuário nunca visto antes
     ├─► Alerta: Volume de DELETEs acima do normal
     └─► Relatório mensal de acessos por usuário (compliance/LGPD)
```

### 3.4 Row Level Security (RLS) — Para dados sensíveis

```sql
-- Restringir acesso a dados por tenant/empresa, sem código na aplicação
ALTER TABLE customer_data ENABLE ROW LEVEL SECURITY;

CREATE POLICY acesso_por_empresa ON customer_data
    USING (empresa_id = current_setting('app.empresa_id')::int);

-- A aplicação define o contexto antes de cada query
SET app.empresa_id = 42;
SELECT * FROM customer_data; -- retorna apenas dados da empresa 42
```

---

## Resultado Esperado: Matriz de Acesso Pós-Implementação

| Perfil | Dev/Staging | Produção (leitura) | Produção (escrita) | DDL em Produção |
|---|---|---|---|---|
| **Desenvolvedor** | Acesso total (dados anonimizados) | Leitura via Bastion (TTL 1h) | ❌ | ❌ |
| **DBA** | Acesso total | Acesso via Bastion + MFA | ✅ (controlado) | ✅ (via pipeline) |
| **Aplicação (runtime)** | — | ✅ (DML apenas) | ✅ (DML apenas) | ❌ |
| **Pipeline CI/CD** | — | ❌ | ❌ | ✅ (apenas migrations aprovadas) |
| **Analista/BI** | Dados anonimizados | ✅ (SELECT only) | ❌ | ❌ |

---

## Como Não Comprometer a Agilidade

A resistência das equipes é real e válida. As contrapartidas práticas:

| Controle implementado | Compensação de agilidade |
|---|---|
| Sem acesso direto de escrita em produção | Ambiente de staging com dados realistas (anonimizados) para debug |
| Migrations via PR | Templates prontos de migration + CI roda em < 5 minutos |
| Aprovação de DDL crítico | SLA de revisão de DDL em 2 horas pelo DBA |
| Acesso via Bastion com MFA | SSO corporativo (um clique + push no celular) |

> **Princípio orientador:** O DBA não é o guardião que diz não. É o parceiro técnico que substitui o acesso arriscado por alternativas seguras e igualmente produtivas. Governança bem implementada é invisível no dia a dia — as equipes trabalham normalmente e o controle acontece nos bastidores.
