# 03 — Arquitetura e Roadmap de Execução

> Documento de planejamento. Guia técnico e sequência de trabalho para os agentes
> de execução. Cada fase é entregável de forma independente e testável.

## 1. Stack e decisões técnicas

| Camada | Escolha | Observação |
|---|---|---|
| Framework | **Next.js (App Router) + TypeScript** | front e back juntos (Route Handlers / Server Actions) |
| Estilo | **Tailwind CSS** | já instalado pelo scaffold |
| ORM | **Prisma** | ver schema no doc 02 |
| Banco | **PostgreSQL** | `numeric` para dinheiro/quantidade |
| Auth | **Auth.js (NextAuth)** com Credentials | sessão carrega `empresaId` e `papel` |
| Validação | **Zod** | validar toda entrada de API/formulário |
| Hash de senha | **bcrypt** (ou argon2) | nunca salvar senha em texto |
| PDF do recibo | **@react-pdf/renderer** ou HTML→print | decidir na Fase 5 |
| Testes | **Vitest** (unit) + **Playwright** (e2e) | Chromium já disponível no ambiente |

### Estado atual do repositório

- ✅ Scaffold `create-next-app` já criado (TypeScript, Tailwind, ESLint, App
  Router, `src/`, alias `@/*`).
- ⛔ Prisma, Auth e dependências **ainda não instalados** (execução parou aqui a
  pedido do cliente — só planejamento nesta etapa).

## 2. Estrutura de pastas proposta

```
src/
  app/
    (auth)/login/                # tela de login
    (app)/                       # área autenticada (layout com sidebar)
      dashboard/
      insumos/
      fornecedores/
      compras/
      produtos/                  # inclui ficha técnica
      producao/
      vendas/                    # PDV + recibo
      relatorios/
    api/                         # Route Handlers (se não usar Server Actions)
  lib/
    db.ts                        # Prisma Client singleton
    tenant.ts                    # helper getTenantDb(empresaId) / extension
    auth.ts                      # config Auth.js
    custeio.ts                   # regras de custo (doc 02, seção 5)
    validators/                  # schemas Zod
  components/                    # UI reutilizável
prisma/
  schema.prisma
  seed.ts                        # empresa demo + usuário OWNER + dados de exemplo
docs/                            # este planejamento
```

## 3. Convenções obrigatórias (para todo agente de execução)

1. **`empresaId` sempre da sessão**, nunca do corpo/query do cliente.
2. **Toda query filtra por `empresaId`** — usar o helper de tenant, não o Prisma cru.
3. **Dinheiro/quantidade = `Decimal`**, nunca `number`/`float`.
4. **Operações de estoque/custo dentro de `prisma.$transaction`.**
5. **Validar entrada com Zod** antes de tocar no banco.
6. **Nunca editar/apagar linhas de `Movimento*`**; corrigir com `AJUSTE`.
7. Textos de UI em **pt-BR**; formatar R$ e datas com `Intl`.
8. Cada fase deve subir com testes do fluxo principal e o app buildar (`next build`).

## 4. Roadmap por fases

Cada item `[ ]` é uma tarefa de execução. As fases são sequenciais; dentro de uma
fase, tarefas podem ser paralelizadas se não dependerem entre si.

### Fase 0 — Fundação
- [ ] Instalar deps: `prisma @prisma/client zod bcryptjs next-auth` (+ dev types).
- [ ] Configurar `DATABASE_URL` (`.env`) e `prisma/schema.prisma` (doc 02).
- [ ] Criar `lib/db.ts` (singleton) e `lib/tenant.ts` (injeção de `empresaId`).
- [ ] `prisma migrate dev` inicial + `prisma/seed.ts` (empresa demo, usuário OWNER,
      alguns insumos/fornecedores de exemplo).
- [ ] Configurar Auth.js (Credentials); sessão expõe `empresaId` e `papel`.
- [ ] Layout autenticado (sidebar) + middleware que protege `(app)/*`.
- **Entregável:** login funcionando, usuário demo entra e vê o dashboard vazio.

### Fase 1 — Cadastros base
- [ ] CRUD **Fornecedores**.
- [ ] CRUD **Insumos** (unidade de compra/uso, fator de conversão, estoque mínimo,
      flag embalagem). Estoque e custos começam em 0.
- [ ] Listagens com busca; validação Zod; proteção por papel (CAIXA não edita).
- **Entregável:** dá para cadastrar insumos e fornecedores da sorveteria.

### Fase 2 — Compras e custo atualizado (DOR #1 e #2)
- [ ] Tela de **registrar compra** (fornecedor, nota, data, itens).
- [ ] Ao salvar: entrada de estoque + atualizar `custoUltimo` e `custoMedio` +
      `MovimentoEstoqueInsumo` (doc 02, seção 5.1), tudo em transação.
- [ ] Histórico de compras e histórico de movimentos por insumo.
- [ ] Configuração de **método de custeio** por empresa.
- **Entregável:** custo do insumo se atualiza sozinho conforme as compras.

### Fase 3 — Produtos e ficha técnica (DOR #4)
- [ ] CRUD **Produtos** (tipo, sabor, unidade de venda, preço).
- [ ] Editor de **ficha técnica**: adicionar insumos e quantidades por lote +
      rendimento.
- [ ] **Custo calculado do produto** exibido em tempo real (doc 02, seção 5.2),
      com margem sugerida sobre o preço.
- **Entregável:** cada sabor mostra quanto custa produzir, baseado nos insumos.

### Fase 4 — Produção
- [ ] Tela **registrar produção** (produto, ficha, nº de lotes).
- [ ] Baixa automática de insumos + geração de produto acabado + custo do lote,
      em transação (doc 02, seção 5.3). Bloquear/avisar se faltar insumo.
- [ ] (Opcional MVP) número de lote e validade.
- **Entregável:** produzir um lote baixa os insumos certos e credita estoque.

### Fase 5 — Vendas e recibo (DOR #3)
- [ ] **PDV simples**: selecionar produtos, quantidade, forma de pagamento.
- [ ] Ao finalizar: baixa produto acabado + gera `Venda`/`ItemVenda` +
      **Recibo numerado por empresa** (doc 02, seção 6), em transação.
- [ ] **Recibo imprimível/PDF** com dados da empresa, itens, total, data, número.
- **Entregável:** vender gera recibo e baixa o estoque de produto acabado.

### Fase 6 — Painel e relatórios
- [ ] Dashboard: estoque de insumo/produto, **alertas de estoque mínimo**.
- [ ] Relatório de **custo x preço x margem** por produto.
- [ ] Relatório de compras e de vendas por período.
- **Entregável:** dono enxerga margem e o que precisa comprar.

### Fase 7+ — Futuro (fora do MVP)
- Fiscal: NFC-e/NF-e; integração com contador.
- Hardware: impressora térmica, balança, gaveta, maquininha/TEF.
- Rastreabilidade de lote/validade e recall.
- Financeiro (contas a pagar/receber, fluxo de caixa, DRE).
- Curva ABC, previsão de demanda/sazonalidade, promoções.
- App mobile / modo offline; delivery / cardápio online.
- Endurecimento multi-tenant com **RLS no PostgreSQL**.

## 5. Ambiente de desenvolvimento (notas)

- PostgreSQL 16 disponível localmente no ambiente (porta 5432). Para dev:
  `DATABASE_URL="postgresql://postgres:postgres@localhost:5432/sorveteria"`.
- Node 22 / npm 10 disponíveis.
- Chromium já instalado (`PLAYWRIGHT_BROWSERS_PATH=/opt/pw-browsers`) — **não**
  rodar `playwright install`.
- **Não commitar** `.env` com segredos; fornecer `.env.example`.

## 6. Definição de pronto (Definition of Done) por fase

Uma fase só está pronta quando:

- `next build` passa sem erros de tipo.
- Existe teste do fluxo principal da fase (unit e/ou e2e).
- Todas as queries novas filtram por `empresaId`.
- Operações de estoque/custo estão em transação.
- UI em pt-BR e valores formatados corretamente.
