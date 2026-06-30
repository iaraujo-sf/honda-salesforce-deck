# Honda - Resumo Executivo das Reuniões

## Reunião 1: MC - DC Honda (09/06/2026)
**Participantes:** Honda (Caio, James), Salesforce (Beatriz/Bia, Bruno, Claudio), Gentrop (Ederson/Jadis, Camila)

### Contexto Apresentado pela Gentrop
- Proposta de migrar segmentações do Marketing Cloud para o Data Cloud (Fase 1 focada nas BUs internas/montadora)
- Linha do tempo do projeto:
  - 2023: Assessment do Marketing Cloud (chaves incorretas, performance)
  - 2024: Squad Marketing Cloud — deduplicação (113M → 70M → 57M contatos atuais)
  - 2025: Squad Data Cloud — motor de oportunidades (propensão de recompra)
  - 2026: Piloto Agent Force SDR (agendamento test drive, ativo em maio, 24/7)

### Arquitetura Proposta
- **Atual:** Site Honda → SDK Data Cloud → Core → Marketing Cloud Engagement + Agent Force
- **Proposta:** Data Cloud assume segmentação (drag & drop, sem SQL), Marketing Cloud Next para novos use cases piloto, fundação de dados para fase 2 (concessionárias)

### Benefícios Prometidos
- Resolver timeouts das SQL queries no Marketing Cloud
- Reduzir dependência do time de TI para segmentações
- Resolução de identidade (perfil unificado = eliminar duplicidades)
- Governança e agilidade nas integrações

### Pushback/Feedback da Honda (Caio e James)

1. **Visão do todo:** Precisam ver a arquitetura completa incluindo o DW e dados transacionais das concessionárias, não só o universo Salesforce
2. **Ceticismo com "ferramenta do momento":** Marketing Cloud foi prometido como "drag & drop simples" e não foi; Marketing Cloud Next está meses sem conseguir setup funcional
3. **Pareto dos problemas está nas concessionárias:** O módulo de campanhas dos dealers (Market One to One) é o maior ofensor em erros de SQL, timeouts, duplicações e volume financeiro (~70% do contrato MC)
4. **DW:** Faz sentido eliminar? Pode substituir pelo Data Cloud? Ou trazer integrações direto? Honda tem também um Data Lake em construção
5. **James mostrou visão macro:** Cliente Honda se relaciona via online (formulários, portal, redes sociais, app Conecte) e offline (concessionárias via DMS → DW → stage → data quality → Core → MC)

### Próximos Passos Acordados (09/06)
- Agendar sessão com **Guilherme** (TI) para deep dive no DW
- Agendar sessão com **Alisson** (CRM) para navegação guiada no módulo de campanhas
- Honda quer entender: DW pode ser eliminado? Campanhas podem ser melhoradas com novos canais (WhatsApp, Push)?
- Beatriz (Bia) vai buscar as agendas e alinhar

---

## Reunião 2: Plano de Trabalho D360 + Campanhas (15/06/2026)
**Participantes internos Salesforce:** Beatriz (Bia), Hamilton, Claudio, Bruno, Paulinha (BP)

### Resumo do Entendimento

#### Frente 1: Eliminar/Reduzir o DW
- **Expectativa Honda (Caio):** "Matar o DW seria ótimo, porque hoje é uma merda" — quer eliminar DW do processo de CRM
- **Realidade (Hamilton):** Data Cloud **não** vai substituir o DW inteiro. Tem muitos dados corporativos, telemetria, etc. Provavelmente ficará no Data Lake
- **Caminho:** Entender quais integrações Core vêm do DW, o que já está no Lake, e o que realmente precisa ir pro Data Cloud
- **Honda já está construindo Data Lake** — pode ser que DW é eliminável combinando Lake + Data Cloud para a parte CRM

#### Frente 2: Módulo de Campanhas dos Dealers
- **O que é:** Customização no Partner Community que simula Distributed Marketing. Dealers fazem disparos de email e SMS pela MC da montadora
- **Problemas:** Timeouts, arquitetura antiga, erros SQL, duplicação
- **Honda quer:** Resolver bugs + adicionar WhatsApp e Push
- **Limitação Salesforce:** Distributed Marketing nativo **não suporta WhatsApp nem Push** (WhatsApp em roadmap distante, Push nem está)
- **Possibilidade discutida:** Provisionar usuários dealer no Data Cloud com políticas de acesso para segmentação, mas consumo de créditos é preocupação
- **Hamilton:** Precisa estressar com time técnico (Zerba) se é viável

### Relação com Gentrop
- Honda está **insatisfeita** com a Gentrop — sente que não têm skills de Core e não conhecem o módulo de campanhas
- Gentrop está contratada para BU montadora, nunca atuou nas concessionárias
- Gentrop não sabia que o principal problema era no módulo de campanhas (pegou mal com Honda)

### Estratégia Comercial
- **Honda terá 9 pessoas no Dreamforce** (setembro), incluindo diretor de CRM
- Contrato Data Cloud atual: modelo Hard Credit até 2030
- Um contrato expira em setembro (créditos já consumidos) — deixar morrer
- Outro contrato expira em 31/janeiro/2027
- **Objetivo:** Negociar contrato grande (ilimitado? Data Cloud + Agent Force + ?) após Dreamforce, fechando em janeiro
- Flex Credit de Agent Force é separado do Data Cloud

### Próximos Passos Definidos (15/06)
1. **Workshop presencial na Honda (Sumaré)** — proposta para 25/06
   - Honda explica DW e módulo de campanhas em detalhe
   - Salesforce ouve, faz perguntas, mapeia
   - Idealmente presencial com todo o time
2. **Envolver arquiteto** (Galvão ou equipe da Lívia/Mitocid) — difícil, depende de aprovação como "deal estratégico"
3. **Preparação:** Honda precisa de briefing do que Salesforce quer ver/entender para se preparar
4. Hamilton precisa assistir a gravação da reunião de 09/06

---

## Pedidos e Expectativas da Honda (Consolidado)

| # | Pedido | Prioridade (Honda) | Complexidade |
|---|--------|-------------------|--------------|
| 1 | Resolver módulo de campanhas (Market One to One) — bugs, timeouts, novos canais (WhatsApp, Push) | ALTA (Pareto) | Alta — sem solução nativa |
| 2 | Eliminar/otimizar DW no fluxo de CRM | ALTA | Média — depende de mapeamento |
| 3 | Visão estratégica/arquitetura completa (não puxadinhos) | ALTA | Média — trabalho consultivo |
| 4 | Unificação de perfis / resolução de identidade | MÉDIA | Média — Data Cloud resolve |
| 5 | Migrar segmentações MC → Data Cloud (montadora) | MÉDIA | Baixa-Média — Gentrop pode executar |
| 6 | Marketing Cloud Next — piloto | BAIXA | Em hold por enquanto |

---

## Para a Reunião de Amanhã (25/06) — Checklist de Preparação

### O que Honda vai apresentar:
- [ ] Deep dive no DW: origem dos dados, regras de qualidade, volumetrias, o que vai pro Core, o que é relevante
- [ ] Navegação guiada no módulo de campanhas: fluxo completo do dealer, segmentação, disparo, erros
- [ ] Envolver Guilherme (TI) e Alisson (CRM)

### O que Salesforce precisa saber/perguntar:
- [ ] Quais dados do DW vão para o Core hoje? Quais são necessários para CRM/campanhas?
- [ ] O Data Lake já está construído? O que já migrou pra lá?
- [ ] No módulo de campanhas: como o dealer faz a segmentação? Que dados usa? Onde dá timeout?
- [ ] Os dados que causam timeout vêm do DW ou de outra fonte?
- [ ] Quantos dealers usam ativamente o módulo? Volume de disparos?
- [ ] Que personalização os dealers fazem? Templates? Canais?
- [ ] Honda aceita piloto com grupo reduzido de concessionárias?

### Decisões pendentes para a reunião:
- [ ] Conseguiu arquiteto (Galvão)?
- [ ] Gentrop participa ou não?
- [ ] É presencial em Sumaré ou remoto?
- [ ] Hamilton confirmou agenda?

### Posicionamento recomendado:
1. **Ouvir primeiro** — não propor soluções nessa sessão, mapear
2. **Não prometer que Data Cloud substitui DW** — posicionar como complemento para CRM
3. **Módulo de campanhas:** ser transparente que não há solução nativa pronta, mas explorar caminhos (Data Cloud + políticas de acesso, Agent para segmentação, etc.)
4. **Mostrar que entende a dor real** (campanhas > jornadas montadora) para reconstruir confiança
5. **Quick wins visíveis** antes de arquitetura perfeita — mas com roadmap claro do todo
