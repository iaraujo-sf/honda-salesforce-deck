# Honda - Análise Técnica e Arquitetural
## Data Cloud, Campanhas e Estratégia de Dados

**Versão:** 1.0  
**Data:** 24/06/2026  
**Classificação:** Interno Salesforce  
**Preparado para:** Workshop Técnico Honda

---

## 1. Situação Atual — Ecossistema de Dados Honda

### 1.1 Visão Geral da Arquitetura Atual

A Honda opera um ecossistema de dados construído ao longo de 10+ anos, com múltiplas camadas de integração e processamento. A arquitetura atual apresenta fragmentação entre sistemas legados e plataformas modernas.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        FONTES DE DADOS - CLIENTE HONDA                       │
├────────────────────────────────┬────────────────────────────────────────────┤
│       ONLINE (Digital)         │           OFFLINE (Transacional)            │
│                                │                                            │
│  • Web2Lead / Leads Connect    │  • DMS (Dealer Management System)          │
│  • Portal do Proprietário      │  • Transações de vendas (2R/4R)            │
│  • Redes Sociais Honda         │  • Serviços e pós-venda                    │
│  • App Conecte (telemetria)    │  • Financeiro concessionárias              │
│  • Formulários de interesse    │  • ~1.000 concessionárias conectadas       │
└────────────────────┬───────────┴──────────────────────┬─────────────────────┘
                     │                                   │
                     ▼                                   ▼
┌────────────────────────────┐    ┌──────────────────────────────────────────┐
│     SALESFORCE CORE        │    │         DATA WAREHOUSE (LEGADO)           │
│     (Sales Cloud)          │◄───│                                          │
│                            │    │  • Staging area (recepção DMS)            │
│  • Contas / Contatos       │    │  • Data Quality (higienização)            │
│  • Oportunidades           │    │  • Deduplicação / Household               │
│  • Cases                   │    │  • Geolocalização                        │
│  • Partner Community       │    │  • Visão Única do Cliente                 │
│    (Market One to One)     │    │  • Regras de acesso por dealer            │
│                            │    │  • Batch processing (não real-time)       │
└──────────┬─────────────────┘    └──────────────────────────────────────────┘
           │                                    │
           │  MC Connect (sync)                 │ (Em construção)
           ▼                                    ▼
┌──────────────────────────┐    ┌──────────────────────────────────────────┐
│  MARKETING CLOUD         │    │           DATA LAKE (NOVO)                │
│  ENGAGEMENT              │    │                                          │
│                          │    │  • Migração gradual do DW                 │
│  • 57M contatos          │    │  • Dados corporativos não-CRM            │
│  • 392 jornadas (running)│    │  • Telemetria                            │
│  • SQL queries (timeout) │    │  • Analytics avançado                    │
│  • 9 BUs consolidadas    │    │                                          │
│  • Campanhas montadora   │    └──────────────────────────────────────────┘
│  • Campanhas dealers     │
└──────────────────────────┘
           │
           ▼
┌──────────────────────────┐
│  DATA CLOUD              │
│                          │
│  • Motor de Propensão    │
│  • SDK Web (navegação)   │
│  • Modelo Hard Credit    │
│  • Identity Resolution   │
│    (parcial)             │
└──────────────────────────┘
           │
           ▼
┌──────────────────────────┐
│  AGENT FORCE SDR         │
│                          │
│  • WhatsApp              │
│  • Test Drive scheduling │
│  • 24/7 (fora horário    │
│    comercial + weekends) │
│  • Produção desde 05/2026│
└──────────────────────────┘
```

### 1.2 Volumetria e Escala

| Métrica | Valor | Observação |
|---------|-------|------------|
| Contatos Marketing Cloud | 57M | Após deduplicação (era 113M em 2023) |
| Concessionárias conectadas | ~1.000 | Impactadas por campanhas dealers |
| Jornadas ativas (running) | 392 | Nem todas necessariamente em produção real |
| BUs Marketing Cloud | 9 | Consolidadas de ~2.000 originais |
| Canais ativos | Email, SMS | WhatsApp via Agent Force (limitado) |
| Canais desejados | WhatsApp, Push | Para módulo de campanhas dealers |
| Contrato Data Cloud | Hard Credit | Até 2030 |
| Modelo propensão | Ativo | Gera oportunidades para dealers |

### 1.3 Fluxo de Dados Atual — Caminho Crítico

```
CONCESSIONÁRIA                 HONDA                          CLIENTE FINAL
     │                           │                                │
     │  DMS (venda/serviço)      │                                │
     ├──────────────────────────►│                                │
     │                           │                                │
     │                    ┌──────┴──────┐                         │
     │                    │     DW      │                         │
     │                    │  (staging)  │                         │
     │                    │  (quality)  │                         │
     │                    │  (dedup)    │                         │
     │                    └──────┬──────┘                         │
     │                           │                                │
     │                    ┌──────┴──────┐                         │
     │                    │    CORE     │                         │
     │                    │(Sales Cloud)│                         │
     │                    └──┬──────┬───┘                         │
     │                       │      │                             │
     │              MC Connect│      │ Data Cloud                  │
     │                       │      │                             │
     │                 ┌─────┴┐  ┌──┴─────┐                      │
     │                 │  MC  │  │   DC   │                       │
     │                 │Engine│  │Propensão│                      │
     │                 └──┬───┘  └────────┘                       │
     │                    │                                       │
     │                    │  Jornada/Campanha                     │
     │                    ├──────────────────────────────────────►│
     │                    │  (Email/SMS)                          │
     │                                                            │
     │  Partner Community                                         │
     │  (Market One to One)                                       │
     ├────────┐                                                   │
     │        │ Segmentação SQL                                   │
     │        │ (TIMEOUT!)                                        │
     │        ▼                                                   │
     │  ┌───────────┐                                             │
     │  │MC Disparo │─────────────────────────────────────────────►
     │  │(Email/SMS)│                                             │
     │  └───────────┘                                             │
```

---

## 2. Diagnóstico dos Problemas Técnicos

### 2.1 Módulo de Campanhas — Market One to One (PRIORIDADE #1)

**O que é:** Customização desenvolvida há ~10 anos no Partner Community que simula um Distributed Marketing. Permite que concessionárias façam disparos segmentados de email e SMS via Marketing Cloud da montadora.

**Fluxo técnico atual:**
1. Dealer acessa Partner Community
2. Seleciona template de campanha
3. Sistema executa SQL queries para segmentação da base do dealer
4. Queries consultam dados sincronizados via MC Connect (Core → MC)
5. Resultados populam Data Extension
6. Disparo executado via Marketing Cloud Engagement

**Problemas identificados:**

| Problema | Causa Raiz | Impacto |
|----------|-----------|---------|
| Timeout nas SQL queries | Queries pesadas sobre 57M contatos; MC não é engine de processamento de dados | Campanhas não executam; erro visível para dealer |
| Duplicação de contatos | Sem resolução de identidade entre canais (app, formulário, concessionária) | Comunicações duplicadas; desperdício de budget |
| Arquitetura obsoleta | Construído há 10 anos; tecnologias evoluíram; customização pesada em Core | Manutenção cara; dependência de pessoas-chave |
| Limitação de canais | Apenas Email e SMS | Não atende demanda de WhatsApp e Push |
| Escalabilidade | ~1.000 dealers executando queries simultâneas | Degradação de performance em horários de pico |

**Impacto financeiro:** ~70% do volume financeiro do contrato Marketing Cloud está associado a este módulo.

**Limitação crítica:** O Distributed Marketing nativo da Salesforce **não suporta WhatsApp nem Push**. WhatsApp está em roadmap futuro (sem data firme) e Push não está no roadmap.

### 2.2 Data Warehouse — Gargalo de Integração (PRIORIDADE #2)

**Função atual do DW:**
- Recebe dados transacionais de ~1.000 concessionárias via DMS
- Aplica regras de Data Quality (higienização, padronização, deduplicação)
- Executa regras de geolocalização e household
- Consolida "Visão Única do Cliente"
- Filtra e envia subconjunto relevante para Salesforce Core
- Processamento em batch (não real-time)

**Problemas:**
- Processamento batch gera latência na atualização dos dados de CRM
- Equipe de CRM não tem acesso direto; depende de TI para qualquer consulta
- Muitas regras legadas que podem não ser mais necessárias
- Custo operacional de manutenção de infraestrutura
- Não alimenta Data Cloud diretamente

**Contexto importante:** Honda já está construindo um Data Lake. O DW não será 100% migrado para Data Cloud — apenas a parte relevante para CRM/Marketing.

### 2.3 Marketing Cloud Connect — Sincronização

**Problema:** Sincronização de objetos entre Core e MC apresenta falhas frequentes. Quando novas jornadas são criadas com novos objetos/dados, a sincronização frequentemente falha.

**Impacto:** Bloqueia a criação de novas jornadas baseadas em dados frescos (telemetria, novos formulários, etc.)

### 2.4 Fragmentação de Identidade

**Situação atual:**
- Cliente compra carro → gera registro via concessionária (ID-A)
- Mesmo cliente baixa app Conecte → gera novo registro (ID-B)
- Mesmo cliente preenche formulário de interesse → gera registro (ID-C)
- IDs não se comunicam entre si

**Consequências:**
- Comunicações duplicadas/triplicadas
- Contexto perdido entre interações (ex: respondeu NPS via ID-A, mas recebe comunicação via ID-B sem esse contexto)
- Impossibilidade de medir efetividade real de canal
- Base inflada artificialmente (57M pode ser significativamente menor em indivíduos únicos)

---

## 3. Arquitetura Proposta — Visão Target

### 3.1 Arquitetura Alvo (Fase Completa)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           FONTES DE DADOS                                    │
├──────────────────────────────┬──────────────────────────────────────────────┤
│     DIGITAL                  │        TRANSACIONAL                           │
│                              │                                              │
│  • SDK Data Cloud (Web)      │  • DMS (via Data Lake ou direta)             │
│  • App Conecte (telemetria)  │  • Integrações concessionárias              │
│  • Formulários              │  • Dados de vendas/serviço                   │
│  • Redes Sociais            │                                              │
└──────────────┬───────────────┴─────────────────────┬────────────────────────┘
               │                                      │
               │         ┌────────────────────┐       │
               │         │    DATA LAKE        │       │
               │         │  (dados não-CRM,    │◄──────┘
               │         │   analytics,        │
               │         │   telemetria full)   │
               │         └─────────┬────────────┘
               │                   │ (subset CRM)
               ▼                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         DATA CLOUD (HUB CENTRAL)                             │
│                                                                              │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐  ┌─────────────────┐   │
│  │  Ingestão   │  │  Modelagem   │  │  Identity   │  │  Segmentação    │   │
│  │  (Streams)  │→ │  (DMO/DLO)   │→ │  Resolution │→ │  (Drag & Drop)  │   │
│  │             │  │              │  │  (Rulesets) │  │  (+ Einstein)   │   │
│  └─────────────┘  └──────────────┘  └─────────────┘  └────────┬────────┘   │
│                                                                 │            │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐           │            │
│  │  Propensão  │  │  Calculated  │  │  Activation │◄──────────┘            │
│  │  (ML Model) │  │  Insights    │  │  Targets    │                        │
│  └─────────────┘  └──────────────┘  └──────┬──────┘                        │
│                                             │                                │
└─────────────────────────────────────────────┼────────────────────────────────┘
                                              │
              ┌───────────────────────────────┼────────────────────┐
              │                               │                    │
              ▼                               ▼                    ▼
┌──────────────────────┐  ┌─────────────────────────┐  ┌────────────────────┐
│  MARKETING CLOUD     │  │  MARKETING CLOUD NEXT   │  │   AGENT FORCE      │
│  ENGAGEMENT          │  │  (Novos Use Cases)      │  │                    │
│                      │  │                         │  │  • SDR (WhatsApp)  │
│  • Jornadas legadas  │  │  • Personalização AI    │  │  • Service Agent   │
│  • Campanhas dealers │  │  • Novos canais         │  │  • 24/7            │
│  • Email / SMS       │  │  • Mensuração avançada  │  │                    │
└──────────────────────┘  └─────────────────────────┘  └────────────────────┘
              │
              │  (Activation do Data Cloud alimenta entrada)
              │
┌─────────────┴───────────────────────────────────────────────────────────────┐
│                      CANAIS DE COMUNICAÇÃO                                    │
│                                                                              │
│    📧 Email    📱 SMS    💬 WhatsApp    🔔 Push    🤖 Agent (conversacional) │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Data Cloud como Hub — Componentes Técnicos

#### Data Streams (Ingestão)
| Fonte | Tipo Conector | Frequência | Volume Estimado |
|-------|--------------|------------|-----------------|
| Salesforce Core | CRM Connector (nativo) | Real-time | Objetos padrão + custom |
| Website Honda | Web SDK (já ativo) | Real-time | Navegação + formulários |
| App Conecte | Mobile SDK / API | Real-time | Telemetria + eventos |
| Data Lake | Cloud Storage Connector | Batch/Scheduled | Subset CRM do DW |
| DMS (futuro) | API / MuleSoft | Near real-time | Transações dealers |

#### Data Model Objects (DMO) — Modelo Proposto

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   Individual     │     │    Vehicle       │     │   Dealership     │
│                  │     │                  │     │                  │
│  • Name          │     │  • Model         │     │  • Dealer Code   │
│  • Email         │◄───►│  • Year          │◄───►│  • Region        │
│  • Phone         │     │  • VIN           │     │  • Group         │
│  • Address       │     │  • Purchase Date │     │  • Channels      │
│  • App ID        │     │  • Service Hist  │     │                  │
│  • Source        │     │  • Telemetry     │     │                  │
└────────┬─────────┘     └──────────────────┘     └──────────────────┘
         │
         │  1:N
         ▼
┌──────────────────┐     ┌──────────────────┐
│  Engagement      │     │  Propensity      │
│                  │     │                  │
│  • Channel       │     │  • Model Score   │
│  • Timestamp     │     │  • Category      │
│  • Campaign ID   │     │  • Confidence    │
│  • Response      │     │  • Calc Date     │
│  • Dealer Ref    │     │  • Target Model  │
└──────────────────┘     └──────────────────┘
```

#### Identity Resolution — Regras Propostas

| Ruleset | Match Criteria | Confiança | Cenário |
|---------|---------------|-----------|---------|
| Rule 1 | CPF (exato) | Alta | Match determinístico direto |
| Rule 2 | Email + Nome (fuzzy) | Alta | Cadastros sem CPF |
| Rule 3 | Phone + Nome (fuzzy) | Média | App + Concessionária |
| Rule 4 | Email + Endereço (parcial) | Média | Formulários web |
| Rule 5 | VIN + Dealer Code | Alta | Veículo como âncora |

**Resultado esperado:** Redução significativa da base de 57M para número real de indivíduos únicos. Estimativa conservadora: 30-40% de redução por unificação.

### 3.3 Solução para Módulo de Campanhas — Opções Técnicas

#### Opção A: Data Cloud + Políticas de Acesso (Data Spaces)

```
┌─────────────────────────────────────────────────────────┐
│                    DATA CLOUD                             │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │ Data Space  │  │ Data Space  │  │ Data Space  │    │
│  │  Dealer A   │  │  Dealer B   │  │  Dealer C   │    │
│  │             │  │             │  │             │    │
│  │ Subset:     │  │ Subset:     │  │ Subset:     │    │
│  │ Clientes    │  │ Clientes    │  │ Clientes    │    │
│  │ região A    │  │ região B    │  │ região C    │    │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘    │
│         │                 │                 │           │
│         └────────────┬────┴────────────────┘           │
│                      ▼                                  │
│         ┌────────────────────────┐                     │
│         │  Segmentation Engine   │                     │
│         │  (per dealer context)  │                     │
│         └───────────┬────────────┘                     │
│                     │                                   │
└─────────────────────┼───────────────────────────────────┘
                      │ Activation Target
                      ▼
┌─────────────────────────────────────────────────────────┐
│  MC ENGAGEMENT (Disparo coordenado pela montadora)       │
│  → Email, SMS, WhatsApp (via Journey), Push (via MC)    │
└─────────────────────────────────────────────────────────┘
```

**Prós:**
- Segmentação nativa do Data Cloud (drag & drop, sem SQL)
- Elimina timeouts de queries
- Escalável para 1.000+ dealers
- Resolução de identidade unificada
- Permite canais adicionais via activation targets

**Contras:**
- Consumo de créditos significativo (1.000 dealers segmentando)
- Complexidade de provisionar Data Spaces por dealer
- Precisa validar viabilidade com equipe técnica DC (Zerba)
- Modelo Hard Credit pode limitar

**Consumo estimado:** Necessita análise com base em volumetria real por dealer.

#### Opção B: Agent Force para Segmentação + MC para Disparo

```
┌─────────────────────────────────────────────────────────┐
│  PARTNER COMMUNITY (Modernizado)                         │
│                                                         │
│  Dealer acessa → Interface de campanha                  │
│       │                                                 │
│       ▼                                                 │
│  ┌──────────────────────────────────┐                   │
│  │  AGENT (Einstein/AgentForce)     │                   │
│  │                                  │                   │
│  │  "Quero enviar campanha de       │                   │
│  │   revisão para clientes que      │                   │
│  │   compraram há 6 meses"          │                   │
│  │                                  │                   │
│  │  → Agent consulta Data Cloud     │                   │
│  │  → Gera segmento automaticamente │                   │
│  │  → Apresenta preview ao dealer   │                   │
│  │  → Dealer aprova e dispara       │                   │
│  └──────────────┬───────────────────┘                   │
│                 │                                        │
└─────────────────┼────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│  DATA CLOUD → Segmentação → Activation → MC Disparo     │
└─────────────────────────────────────────────────────────┘
```

**Prós:**
- UX moderna e intuitiva para dealer
- Elimina complexidade de SQL
- Segmentação guiada por AI
- Dealer não precisa conhecer estrutura de dados
- Experiência conversacional (Honda já usa Agent Force)

**Contras:**
- Desenvolvimento custom necessário
- Maturidade do Agent Force para este use case
- Governança: garantir que dealer não acesse dados de outro
- Ainda depende de MC para disparo multicanal

#### Opção C: Híbrida (Recomendada para Fase 1)

```
FASE 1 (Quick Win - 3 meses):
├── Data Cloud assume segmentação das jornadas montadora
├── Resolve timeouts (SQL eliminado)
├── Identity Resolution ativada
└── Módulo dealers: mantém fluxo atual com fix pontual nos timeouts

FASE 2 (6 meses):
├── Piloto módulo dealers com 5-10 concessionárias
├── Segmentação via Data Cloud com políticas de acesso
├── Novos canais (WhatsApp) via Journey trigger
└── Validação de consumo de créditos

FASE 3 (12 meses):
├── Rollout para todas concessionárias
├── Agent Force como interface de segmentação para dealers
├── Eliminação do módulo legado Partner Community
├── Integração DMS → Data Cloud (bypass DW para dados CRM)
└── Marketing Cloud Next para use cases avançados
```

---

## 4. Análise de Integração — DW vs Data Cloud

### 4.1 Mapeamento de Capacidades

| Capacidade DW | Data Cloud Equivalente | Viabilidade | Observação |
|---------------|----------------------|-------------|------------|
| Recepção DMS (staging) | Data Streams (Ingestion API) | Alta | Conector API ou file-based |
| Data Quality (higienização) | Data Prep / Formulas / Transforms | Média | Nem todas as regras têm equivalente |
| Deduplicação | Identity Resolution (Rulesets) | Alta | Potencialmente superior ao DW |
| Geolocalização | Calculated Insights / Formulas | Média | Depende da complexidade atual |
| Household rules | Identity Resolution (Household) | Alta | Feature nativa do DC |
| Visão Única Cliente | Unified Profile | Alta | Core capability do DC |
| Filtro para Core | Data Actions / CRM Writeback | Alta | Bi-direcional com Core |
| Batch processing | Streaming + Scheduled Segments | Alta | Ganho: real-time vs batch |
| Regras de acesso dealer | Data Spaces / Permission Sets | Média | Precisa validar escala |

### 4.2 O Que NÃO Migra para Data Cloud

- Dados corporativos não-CRM (financeiro, fiscal, operacional)
- Analytics pesado (BI, dashboards operacionais)
- Integrações com sistemas não-Salesforce que não são de CRM
- Histórico completo transacional (volume vs custo de créditos)

**Destino:** Data Lake (já em construção pela Honda)

### 4.3 Modelo de Transição

```
ESTADO ATUAL:
  DMS → DW (todas as regras) → Core → MC

ESTADO INTERMEDIÁRIO (Fase 1-2):
  DMS → DW → Core → Data Cloud (via CRM Connector) → Segmentação → MC
  DMS → DW → Data Lake (dados não-CRM)

ESTADO ALVO (Fase 3):
  DMS → Data Cloud (dados CRM) → Core (writeback seletivo) → MC/Next/Agent
  DMS → Data Lake (dados não-CRM, analytics)
  DW: DESCOMISSIONADO
```

---

## 5. Consumo e Licenciamento

### 5.1 Situação Contratual Atual

| Item | Detalhe |
|------|---------|
| Data Cloud | Hard Credit até 2030 |
| Contrato 1 (MC?) | Expira setembro/2026 — créditos já consumidos |
| Contrato 2 | Expira 31/janeiro/2027 |
| Agent Force | Flex Credit (separado do DC) |

### 5.2 Cenários de Consumo Data Cloud

**Cenário conservador (apenas montadora):**
- Segmentação de jornadas internas
- Identity Resolution sobre base unificada
- Activation para MC Engagement
- **Impacto:** Dentro do envelope atual (Hard Credit)

**Cenário expandido (montadora + dealers):**
- Tudo acima +
- Data Spaces por dealer (ou cluster de dealers)
- Segmentação por dealer
- Activation multicanal
- **Impacto:** Possivelmente excede envelope atual → necessário upgrade ou modelo ilimitado

### 5.3 Estratégia Comercial

**Objetivo pós-Dreamforce (setembro/2026):** Negociar contrato consolidado e expandido, possivelmente modelo ilimitado (Data Cloud + Agent Force + MC), com fechamento em janeiro/2027 (alinhado com expiração do contrato 2).

---

## 6. Riscos e Mitigações

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Data Cloud não resolve timeouts de dealers | Baixa | Alto | POC com subset antes de rollout |
| Consumo de créditos explode com dealers | Média | Alto | Piloto com 5-10 dealers; medir antes de expandir |
| Complexidade de regras DW não replicável | Média | Médio | Mapear regras; aceitar que algumas ficam no Lake |
| Honda perde paciência com mais promessas | Alta | Alto | Quick wins visíveis em 30-60 dias |
| Gentrop não consegue executar parte Core | Alta | Médio | Avaliar se Salesforce assume ou outro parceiro |
| Marketing Cloud Next não está pronto | Média | Baixo | Manter como piloto secundário; não é entregável principal |
| Arquiteto indisponível | Alta | Médio | Escalar via deal estratégico; envolver Galvão/Mitocid |

---

## 7. Perguntas-Chave para Workshop (25/06)

### Para Guilherme (TI — DW):
1. Quais integrações DMS → DW existem? Protocolos (API, file, SFTP)?
2. Qual a frequência de atualização dos dados? Diária? Horária?
3. Quantas regras de Data Quality estão ativas? Documentação existe?
4. Quais dados do DW vão para o Core hoje? Todos ou subset?
5. O Data Lake já recebe dados do DMS diretamente ou só do DW?
6. Qual tecnologia do DW? (SQL Server, Oracle, Teradata?)
7. Volume diário de registros novos/atualizados?
8. Regras de geolocalização e household: são críticas para CRM ou para outros usos?

### Para Alisson (CRM — Módulo Campanhas):
1. Demonstração completa do fluxo: dealer seleciona → segmenta → dispara
2. Em que ponto exatamente ocorre o timeout? Na query de segmentação?
3. Quantos dealers usam ativamente o módulo por mês?
4. Volume médio de disparos por dealer/mês?
5. Que dados o dealer usa para segmentar? (modelo, ano compra, região, etc.)
6. Dealer personaliza conteúdo ou usa templates fixos?
7. Existem regras de governança? (ex: não mais que X disparos/mês por cliente)
8. WhatsApp e Push: qual o use case desejado? Mesmo fluxo ou novo?
9. Quais são os erros mais frequentes além de timeout?
10. O módulo depende de objetos custom no Core? Quais?

### Para Ambos:
1. Quando um cliente troca de concessionária, como isso se reflete nos dados?
2. Base por dealer: como é definida? Geolocalização? Última compra? Manual?
3. Existe sobreposição de base entre dealers? Conflitos de ownership?

---

## 8. Roadmap Proposto (Sujeito a Validação no Workshop)

```
JUL 2026          AGO 2026          SET 2026          OUT-DEZ 2026       JAN 2027
    │                 │                 │                  │                 │
    ▼                 ▼                 ▼                  ▼                 ▼
┌────────┐      ┌────────┐      ┌────────────┐     ┌──────────┐     ┌──────────┐
│MAPEAR  │      │FASE 1  │      │DREAMFORCE  │     │FASE 2    │     │CONTRATO  │
│        │      │EXECUÇÃO│      │            │     │          │     │          │
│• DW    │      │        │      │• Demo DC   │     │• Piloto  │     │• Fecho   │
│  deep  │      │• Seg.  │      │  em ação   │     │  dealers │     │  contrato│
│  dive  │      │  MC→DC │      │• ROI       │     │  (5-10)  │     │  ilimit. │
│• Módulo│      │  (mont)│      │  projetado │     │• WhatsApp│     │• Escopo  │
│  camp. │      │• ID    │      │• Visão     │     │  canal   │     │  2027    │
│• Arch. │      │  Resol.│      │  estratég. │     │• Agent   │     │          │
│  design│      │• Quick │      │            │     │  Force   │     │          │
│        │      │  Win   │      │            │     │  POC     │     │          │
└────────┘      └────────┘      └────────────┘     └──────────┘     └──────────┘
```

---

## 9. Anexo — Comparativo de Performance (Case Referência)

Dados compartilhados de cliente do setor farmacêutico (~3M contatos, múltiplas orgs) que migrou segmentação para Data Cloud:

| Métrica | Antes (MC Only) | Depois (Data Cloud) | Melhoria |
|---------|-----------------|--------------------:|----------|
| Tempo de segmentação | 45-60 min | 2-5 min | ~90% |
| Taxa de erro (governança) | 12% | <1% | ~95% |
| Tempo atualização dados | 24h (batch) | Near real-time | Significativa |
| Dependência TI para segmentos | 100% | 20% | 80% redução |
| Tempo ativação campanha | 4-6h | 30 min | ~90% |

*Nota: Resultados variam por implementação. Honda tem escala significativamente maior (57M vs 3M).*

---

**Documento preparado por equipe Salesforce para uso interno. Conteúdo baseado em reuniões de 09/06/2026 e 15/06/2026.**
