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
que liga **compra/importação de NF → custo real → calda base → ficha técnica por
sabor → produção (com lote/validade) → estoque → venda (bola, kg, pote, balde...) →
recibo → margem**, resolvendo as seis dores centrais do cliente.

## Dores cobertas pelo plano

1. Controle de estoque (com **lote e validade**).
2. Custo atualizado pelas compras — inclui **custo da calda base** + produto acabado.
3. Emissão de recibo.
4. Divisão de insumos por sabor/tipo (**ficha técnica multi-nível**).
5. **Entrada de estoque por nota fiscal** (XML da NF-e + foto/OCR), com mapeamento
   automático dos produtos.
6. **Preços praticados e margem** por produto/formato.

## Estado atual

- ✅ Stack e escopo comercial decididos com o cliente.
- ✅ Scaffold `create-next-app` criado no repositório.
- ✅ Planejamento documentado e revisado com as decisões do cliente (esta pasta).
- ⏭️ Próximo passo: executar a **Fase 0** do roadmap (doc 03).
