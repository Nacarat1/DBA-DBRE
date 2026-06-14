# Desafio Técnico — Analista de Banco de Dados (DBA / DBRE)

Repositório com as respostas ao desafio técnico para a posição de **Analista de Banco de Dados**. 

## Índice

| # | Tema | Link |
|---|---|---|
| 01 | Assumindo um Ambiente Crítico | [questao-01.md](./questao-01.md) |
| 02 | Atualização sem Impacto ao Negócio | [questao-02.md](./questao-02.md) |
| 03 | Diagnóstico de Lentidão em Produção | [questao-03.md](./questao-03.md) |
| 04 | Segurança e Governança de Banco de Dados | [questao-04.md](./questao-04.md) |
| 05 | Automação e Uso de Inteligência Artificial | [questao-05.md](./questao-05.md) |

---

## Questão 01 — Assumindo um Ambiente Crítico

### Requisitos de Negócio
- Garantir a continuidade de sistemas críticos da empresa suportados pelo ambiente PostgreSQL
- Estabelecer segurança mínima dos dados em um ambiente desconhecido e sem histórico documentado
- Identificar riscos imediatos antes de qualquer intervenção

### Restrições e Prioridades Identificadas
- **Restrição crítica:** ausência de backup validado, qualquer falha de hardware ou erro humano pode resultar em perda total e irrecuperável de dados
- **Restrição operacional:** sem documentação de arquitetura, topologia ou histórico de configurações
- **Prioridade máxima:** proteção dos dados antes de qualquer diagnóstico de performance ou processo de documentação

### Metodologia Aplicada
A abordagem foi estruturada como **resposta a incidente ativo**, não como onboarding padrão. O princípio orientador: um ambiente sem backup validado é um ambiente em risco e toda investigação secundária é bloqueada até essa condição ser resolvida. 
As 72 horas foram divididas em fases com critérios claros de avanço: proteção → validação → diagnóstico → estruturação → documentação.

### Soluções Apresentadas
- **Backup físico imediato** com `pg_basebackup` e WAL streaming, executado nas primeiras 2 horas
- **Validação de restore** em sandbox isolado (Docker) com comparação de row counts entre produção e restore
- **Mapeamento do ambiente:** topologia, configurações críticas, saúde de tabelas, locks e controle de acessos
- **Implementação de backup gerenciado** com pgBackRest (backups incrementais, retenção automática, verificação de integridade)
- **Alertas operacionais mínimos:** espaço em disco, lag de replicação, falha de arquivamento de WAL
- **Checklist de evidências auditáveis** para declarar o ambiente minimamente seguro

---

## Questão 02 — Atualização sem Impacto ao Negócio

### Requisitos de Negócio
- Migrar o ambiente PostgreSQL de uma versão próxima ao fim de suporte para uma versão atual
- Garantir a continuidade de serviço para milhares de usuários simultâneos durante a migração
- Eliminar o risco de exposição a vulnerabilidades de segurança sem patches após o fim do suporte

### Restrições e Prioridades Identificadas
- **Restrição de disponibilidade:** ambiente com alto volume de usuários e tolerância mínima a downtime
- **Restrição técnica:** possível incompatibilidade de extensões, sintaxes depreciadas e drivers de aplicação
- **Prioridade:** a estratégia de migração deve ser escolhida com base na tolerância real a downtime — não existe uma única abordagem correta para todos os cenários

### Metodologia Aplicada
O upgrade foi tratado como **projeto estruturado em fases**, com antecedência mínima de 12 semanas. A metodologia inclui: diagnóstico de compatibilidade (`pg_upgrade --check`), testes em ambiente de homologação idêntico à produção com dados reais anonimizados, e seleção da estratégia de migração conforme a janela de downtime disponível. O plano de rollback é definido e testado antes da execução em produção.

### Soluções Apresentadas
- **Diagnóstico prévio** com `pg_upgrade --check` e inventário de extensões instaladas
- **Três estratégias de upgrade** mapeadas por tolerância a downtime:
  - *Replicação Lógica*: downtime de segundos — dados migram em tempo real enquanto a versão antiga opera
  - *pg_upgrade na réplica + failover*: downtime de 1 a 5 minutos — upgrade no standby e promoção controlada
  - *pg_upgrade --link no primary*: downtime de 30 a 60 minutos — para ambientes com janela de manutenção
- **Plano de rollback** com critérios claros de ativação e tempo de recuperação por estratégia
- **Pós-upgrade obrigatório:** `ANALYZE`, `REINDEX` e atualização de extensões para restaurar performance do planner
- **Timeline de 12 semanas** com marco de validação em homologação antes de execução em produção

---

## Questão 03 — Diagnóstico de Lentidão em Produção

### Requisitos de Negócio
- Identificar e resolver a causa da lentidão reportada por usuários após um novo deploy
- Minimizar o tempo de impacto ao negócio com investigação estruturada e objetiva
- Documentar a causa raiz para evitar recorrência

### Restrições e Prioridades Identificadas
- **Contexto determinante:** a lentidão começou após um deploy específico — isso filtra as hipóteses e concentra a investigação no que mudou
- **Restrição de tempo:** em produção com usuários afetados, cada minuto de investigação mal direcionado agrava o impacto
- **Prioridade:** identificar se é problema de banco, aplicação ou infraestrutura antes de agir — ação equivocada pode piorar o cenário

### Metodologia Aplicada
A investigação segue **eliminação de hipóteses por camada**, da mais provável (banco, dado o contexto de deploy) para as demais (infraestrutura e aplicação). O processo começa com perguntas de escopo para diferenciar lentidão generalizada de localizada — o que já direciona a ferramenta certa. Cada camada tem seus próprios indicadores e ferramentas de diagnóstico.

### Soluções Apresentadas
- **Triagem inicial em minutos:** `pg_stat_activity` para identificar conexões bloqueadas, queries longas e pool esgotado
- **Diagnóstico de banco:** `pg_stat_statements` para queries mais lentas, análise de lock chains, `EXPLAIN (ANALYZE, BUFFERS)` para planos reais de execução
- **Causas mais comuns pós-deploy** mapeadas: índice removido por migration, `idle in transaction` por transação não encerrada, estatísticas desatualizadas após carga de dados
- **Diagnóstico de infraestrutura:** `iostat`, `vmstat` e CPU steal para gargalos de I/O, memória e CPU
- **Tabela de diferenciação** de sintomas por camada (banco vs aplicação vs infra)
- **Soluções operacionais:** `CREATE INDEX CONCURRENTLY` para criar índices sem bloqueio, `ANALYZE` para atualizar estatísticas, `pg_terminate_backend` criterioso para locks abandonados

---

## Questão 04 — Segurança e Governança de Banco de Dados

### Requisitos de Negócio
- Implementar controles de segurança em ambiente onde desenvolvedores têm acesso direto ao banco de produção
- Garantir rastreabilidade completa de quem fez o quê e quando
- Aumentar a segurança sem comprometer a agilidade e produtividade das equipes

### Restrições e Prioridades Identificadas
- **Restrição cultural:** controles que tornam o trabalho mais lento que o acesso direto atual serão boicotados — as alternativas precisam ser igualmente ágeis ou melhores
- **Restrição técnica:** alterações de schema sem processo formal representam risco contínuo de perda de dados e inconsistências
- **Prioridade:** o acesso direto de escrita em produção deve ser removido apenas após as alternativas estarem funcionando — não antes

### Metodologia Aplicada
A governança foi estruturada em **3 pilares complementares**: controle de quem acessa (identidade e privilégios), controle de como as mudanças acontecem (processo de aprovação e deploy), e controle do que aconteceu (auditoria e rastreabilidade). A implementação segue uma timeline de 6 semanas, garantindo que cada controle tem sua alternativa operacional antes de ser ativado.

### Soluções Apresentadas
- **Controle de acessos com RBAC:** roles funcionais por perfil (app, readonly, DBA, pipeline), princípio do menor privilégio, remoção de superusers desnecessários
- **Acesso via Bastion/Vault:** credenciais temporárias com TTL, autenticação MFA + SSO, session recording — elimina senhas fixas e acesso anônimo
- **Mascaramento de dados em staging:** desenvolvedores trabalham com dados realistas sem expor informações pessoais (conformidade com LGPD)
- **Migrations versionadas via CI/CD:** Flyway/Liquibase, aprovação por nível de risco da alteração, DDL apenas pelo pipeline — elimina alterações manuais em produção
- **pgAudit:** auditoria de DDL, DCL e conexões com registro do objeto afetado — logs centralizados em SIEM com alertas para alterações fora do processo
- **Row Level Security:** filtro de dados por contexto diretamente no banco, sem dependência de WHERE na aplicação

---

## Questão 05 — Automação e Uso de Inteligência Artificial

### Requisitos de Negócio
- Aumentar produtividade operacional eliminando trabalho manual repetitivo
- Reduzir riscos operacionais com detecção proativa de problemas
- Melhorar a qualidade da administração com uso responsável de tecnologias emergentes

### Restrições e Prioridades Identificadas
- **Restrição de segurança:** dados reais e schemas de produção não devem ser enviados para LLMs públicos — risco de privacidade e LGPD
- **Restrição de confiabilidade:** ações irreversíveis em produção (DROP, ALTER TABLE, pg_terminate) não devem ser executadas por automação sem supervisão humana
- **Prioridade:** distinguir claramente o que é seguro automatizar completamente do que exige julgamento humano antes de executar

### Metodologia Aplicada
A abordagem diferencia **automação determinística** (scripts testados, comportamento previsível) de **uso de IA como acelerador** (geração de hipóteses, não de ações). Toda recomendação de IA passa por validação em 3 etapas: entender o mecanismo, testar em staging, monitorar após aplicar em produção. Automação plena é reservada para operações repetitivas, previsíveis e de baixo risco.

### Soluções Apresentadas
- **Validação automática de backup:** script semanal que executa restore completo em sandbox efêmero, valida integridade e notifica o resultado — elimina o risco de backup nunca testado
- **Vacuum advisor automatizado:** Python + psycopg2 que monitora bloat por tabela e executa `VACUUM ANALYZE` automaticamente em tabelas pequenas com bloat crítico, alertando para tabelas grandes
- **Infrastructure as Code:** Terraform para provisionamento padronizado de instâncias PostgreSQL com parâmetros otimizados, backup e monitoramento configurados no próprio código
- **IA aplicada:** análise de `EXPLAIN ANALYZE` complexos, otimização de queries analíticas, ferramentas especializadas (pganalyze para index advisor, Datadog para anomaly detection)
- **Tabela de limites:** o que automatizar completamente vs o que manter sob controle humano — critério baseado em reversibilidade e amplitude de impacto

---

## Sobre a Abordagem Geral
 O foco em cada questão foi o **raciocínio por trás das decisões** e o critério usado para escolher entre alternativas, priorizar ações e gerenciar riscos em cenários com incerteza.
