# Planejamento — Sistema para Sorveteria / Fábrica de Sorvete

Esta pasta contém o **planejamento** do projeto. Nesta etapa **não há execução de
features** — o objetivo é deixar as decisões e o roadmap prontos para que a
implementação seja feita depois.

Leia nesta ordem:

1. **[01 — Visão, Público e Escopo](./01-visao-e-escopo.md)**
   O que o sistema faz, para quem, glossário do domínio, escopo do MVP e decisões
   em aberto.
2. **[02 — Modelo de Dados e Multi-Tenancy](./02-modelo-de-dados.md)**
   Estratégia multi-empresa, schema Prisma de referência e as regras de custo e
   estoque (o núcleo do valor do sistema).
3. **[03 — Arquitetura e Roadmap de Execução](./03-arquitetura-e-roadmap.md)**
   Stack, estrutura de pastas, convenções obrigatórias e o roadmap por fases.

## Resumo em uma frase

Web app **multi-empresa** (Next.js + TypeScript + Prisma + PostgreSQL + Tailwind)
que liga **compra de insumo → custo real → ficha técnica → produção → estoque →
venda → recibo → margem**, resolvendo as quatro dores centrais: controle de
estoque, custo atualizado, recibo e divisão de insumos por sabor/tipo.

## Estado atual

- ✅ Stack e escopo comercial decididos com o cliente.
- ✅ Scaffold `create-next-app` criado no repositório.
- ✅ Planejamento documentado (esta pasta).
- ⏭️ Próximo passo: executar a **Fase 0** do roadmap (doc 03).
