# SysSorv — Sistema para Sorveteria / Fábrica de Sorvete

SaaS **multi-empresa** para sorveteria que também é fábrica de sorvete. Liga
**compra/importação de NF → custo real (calda base + produto) → ficha técnica →
produção (lote/validade) → estoque → venda (bola, kg, pote, balde, sundae...) →
recibo → margem**.

> **Status:** etapa de **planejamento**. O código é o scaffold inicial; as
> funcionalidades serão implementadas seguindo o roadmap em [`docs/`](./docs).
> Comece por [`docs/README.md`](./docs/README.md).

## Stack

Next.js (App Router) + TypeScript · Prisma + PostgreSQL · Tailwind CSS · Auth.js ·
Zod. Detalhes em [`docs/03-arquitetura-e-roadmap.md`](./docs/03-arquitetura-e-roadmap.md).

## Rodando localmente

Pré-requisitos: **Node 22+** e **Git**.

```bash
# 1. Clonar (de preferência FORA de pasta sincronizada do Google Drive)
git clone https://github.com/ademirds-dev/sistema-sorveteria.git SysSorv
cd SysSorv
git checkout claude/ice-cream-shop-system-e4bd1u

# 2. Instalar dependências
npm install

# 3. Rodar em desenvolvimento
npm run dev
```

Abra <http://localhost:3000>.

Scripts disponíveis:

| Comando | O que faz |
|---|---|
| `npm run dev` | Servidor de desenvolvimento |
| `npm run build` | Build de produção |
| `npm run start` | Sobe o build de produção |
| `npm run lint` | ESLint |

## Variáveis de ambiente

Copie `.env.example` para `.env` e ajuste. O `.env` **não** é versionado.

```bash
cp .env.example .env
```

> As variáveis de banco (`DATABASE_URL`) passam a ser usadas a partir da **Fase 0**
> do roadmap, quando o Prisma/PostgreSQL forem configurados. No scaffold atual ainda
> não são necessárias para `npm run dev`.

## Documentação de planejamento

- [`docs/01-visao-e-escopo.md`](./docs/01-visao-e-escopo.md) — visão, escopo do MVP, decisões.
- [`docs/02-modelo-de-dados.md`](./docs/02-modelo-de-dados.md) — modelo multi-tenant e regras de custo/estoque.
- [`docs/03-arquitetura-e-roadmap.md`](./docs/03-arquitetura-e-roadmap.md) — stack, convenções e roadmap por fases.
- [`docs/04-analise-concorrencia.md`](./docs/04-analise-concorrencia.md) — análise de mercado e posicionamento.

## Convenção do repositório

Veja [`AGENTS.md`](./AGENTS.md): este projeto usa uma versão do Next.js com possíveis
mudanças de API — consulte os guias em `node_modules/next/dist/docs/` antes de
escrever código.

## Git

Trabalho na branch `claude/ice-cream-shop-system-e4bd1u`. Fluxo:

```bash
git pull            # antes de começar
# ... edições ...
git add -A && git commit -m "mensagem" && git push
```
