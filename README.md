# Honda × Salesforce — Data360 Hub Strategy

Apresentação executiva da estratégia de modernização de dados Honda com Salesforce Data Cloud.

## 🚀 Deploy

**Produção**: https://honda-lovat-gamma.vercel.app

Hospedado na Vercel com deploy automático do branch `main`.

## 📋 Estrutura do Projeto

```
honda/
├── deck/                    # Apresentação principal
│   ├── index.html          # Versão de produção
│   ├── index-v3.html       # Versão de desenvolvimento
│   ├── responsive-test.html # Testes de responsividade
│   └── assets/             # Imagens e recursos
├── docs/                    # Documentação
└── transcricoes/           # Transcrições de reuniões
```

## 🎯 Slides

1. **Diagrama de Arquitetura** - Visão técnica em 5 camadas
2. **Situação Atual** - Métricas e bloqueadores
3. **Problemas Críticos** - 4 bloqueadores P0
4. **Arquitetura Proposta** - 5 mudanças estruturais
5. **Roadmap Executivo** - 6 ondas de migração
6. **Benefícios Mensuráveis** - 8 KPIs de impacto
7. **Próximos Passos** - 3 ações imediatas
8. **Obrigado** - Encerramento

## 💻 Desenvolvimento Local

```bash
# Abrir apresentação localmente
open deck/index.html

# Ou usar servidor HTTP
python3 -m http.server 8000
# Acesse http://localhost:8000/deck/
```

## 🎨 Design

- **Responsivo**: Otimizado para resoluções 1440px+
- **Brand**: Cores Honda (vermelho #CC0000) + Salesforce Data Cloud (roxo #9333EA)
- **Navegação**: Teclado (← →), clique, scroll, ou dots na parte inferior

## 📊 Analytics

Google Analytics configurado. Substitua `G-XXXXXXXXXX` no `index.html` pelo ID correto.

## 🔄 Histórico de Versões

- **v3.0** (2026-06-30): Responsivo completo para 1440px+ com viewport units
- **v2.0**: Estrutura de 8 slides com animações
- **v1.0**: Versão inicial

## 📝 Notas

- Apresentação otimizada para projeção em salas de reunião
- Conteúdo baseado em reuniões presenciais e análise técnica
- Dados atualizados em junho/2026
