# 03 — Arquitetura e Roadmap de Execução

> Documento de planejamento. Guia técnico e sequência de trabalho para os agentes
> de execução. Cada fase é entregável de forma independente e testável.

## 1. Stack e decisões técnicas

| Camada | Escolha | Observação |
|---|---|---|
| Framework | **Next.js (App Router) + TypeScript** | front e back juntos |
| Estilo | **Tailwind CSS** | já instalado pelo scaffold |
| ORM | **Prisma** | ver schema no doc 02 |
| Banco | **PostgreSQL** | `numeric` para dinheiro/quantidade |
| Auth | **Auth.js (NextAuth)** Credentials | sessão carrega `empresaId` e `papel` |
| Validação | **Zod** | validar toda entrada |
| Hash de senha | **bcrypt** (ou argon2) | |
| Parse de XML NF-e | **fast-xml-parser** | ler `det`/`emit` do XML |
| OCR da nota (foto) | **modelo de visão** (ex.: Claude via API Anthropic) | ver seção 5 |
| PDF do recibo | **@react-pdf/renderer** ou HTML→print | decidir na Fase 6 |
| Testes | **Vitest** (unit) + **Playwright** (e2e) | Chromium já disponível |

### Estado atual do repositório
- ✅ Scaffold `create-next-app` (TypeScript, Tailwind, ESLint, App Router, `src/`, `@/*`).
- ⛔ Prisma, Auth e demais deps **ainda não instalados** (etapa atual é só planejamento).

## 2. Estrutura de pastas proposta

```
src/
  app/
    (auth)/login/
    (app)/
      dashboard/
      itens/                     # cadastro unificado (insumo/intermediário/produto)
      fornecedores/
      compras/
        importar-nf/             # XML e foto
      receitas/                  # ficha técnica multi-nível (calda base + sabores)
      producao/
      vendas/                    # PDV + formatos + recibo
      relatorios/
    api/
  lib/
    db.ts                        # Prisma singleton
    tenant.ts                    # getTenantDb(empresaId) / extension
    auth.ts
    custeio.ts                   # roll-up de custo (doc 02, seção 5)
    estoque.ts                   # regras de estoque/FEFO (doc 02, seção 6)
    nfe/
      parseXml.ts                # parse NF-e
      ocr.ts                     # extração por foto
      mapeamento.ts              # produto-fornecedor -> insumo
    validators/                  # schemas Zod
  components/
prisma/
  schema.prisma
  seed.ts                        # empresa demo, usuário OWNER, insumos, calda base
docs/
```

## 3. Convenções obrigatórias (para todo agente de execução)

1. **`empresaId` sempre da sessão**, nunca do cliente.
2. **Toda query filtra por `empresaId`** — usar o helper de tenant.
3. **Dinheiro/quantidade = `Decimal`**, nunca `number`/`float`.
4. **Operações de estoque/custo/produção/venda dentro de `prisma.$transaction`.**
5. **Validar entrada com Zod** antes de tocar no banco.
6. **Nunca editar/apagar `MovimentoEstoque`**; corrigir com `AJUSTE`.
7. **Baixa por lote = FEFO** (vence primeiro sai primeiro), salvo decisão contrária.
8. **Custo em cascata** via `lib/custeio.ts`; detectar ciclo na BOM.
9. Textos de UI em **pt-BR**; formatar R$ e datas com `Intl`.
10. Cada fase sobe com testes do fluxo principal e `next build` passando.

## 4. Roadmap por fases

### Fase 0 — Fundação
- [ ] Instalar deps: `prisma @prisma/client zod bcryptjs next-auth fast-xml-parser`.
- [ ] `.env` (`DATABASE_URL`) + `prisma/schema.prisma` (doc 02).
- [ ] `lib/db.ts` (singleton) e `lib/tenant.ts` (injeção de `empresaId`).
- [ ] `prisma migrate dev` inicial + `prisma/seed.ts` (empresa demo, OWNER, alguns
      insumos, uma calda base de exemplo).
- [ ] Auth.js (Credentials); sessão expõe `empresaId` e `papel`.
- [ ] Layout autenticado (sidebar) + middleware protegendo `(app)/*`.
- **Entregável:** login funcionando; usuário demo entra e vê o dashboard.

### Fase 1 — Cadastros base
- [ ] CRUD **Fornecedores**.
- [ ] CRUD **Itens** unificado (INSUMO / INTERMEDIÁRIO / PRODUTO): unidade de
      estoque/compra, fator de conversão, estoque mínimo, flags embalagem e lote.
- [ ] Listagens com busca; proteção por papel (CAIXA não edita cadastro).
- **Entregável:** cadastrar insumos, calda base (como item) e produtos.

### Fase 2 — Compras e custo (DOR #1, #2)
- [ ] Tela **registrar compra** manual (fornecedor, nota, itens).
- [ ] Ao salvar: entrada de estoque + `custoUltimo`/`custoMedio` + `MovimentoEstoque`
      + `Lote` quando aplicável, em transação (doc 02, 6.1).
- [ ] Histórico de compras e de movimentos por item.
- [ ] Configuração de **método de custeio** por empresa.
- **Entregável:** custo do insumo se atualiza sozinho conforme as compras.

### Fase 3 — Importação de NF (DOR #5)
- [ ] **Import XML NF-e**: upload → parse (`fast-xml-parser`) → cria
      `NotaFiscalImportada` + itens → auto-mapeia via `MapeamentoInsumoFornecedor`.
- [ ] Tela de **revisão/mapeamento**: mapear item da NF → `Item` (INSUMO) + fator
      de conversão; **salva o mapeamento** para as próximas notas.
- [ ] **Confirmar** a nota gera a compra e a entrada de estoque (reusa Fase 2).
- [ ] **Import por foto (OCR)**: upload de imagem → extração dos itens → mesmo fluxo
      de revisão (revisão humana obrigatória).
- **Entregável:** dar entrada em nota por XML/foto sem redigitar itens.

### Fase 4 — Receitas multi-nível e custo em cascata (DOR #4, parte da #2)
- [ ] Editor de **ficha técnica** para INTERMEDIÁRIO (calda base) e PRODUTO,
      adicionando componentes (insumos e/ou a calda base) + rendimento.
- [ ] **Custo em cascata** exibido em tempo real: custo da calda base e custo do
      produto acabado (doc 02, seção 5), com detecção de ciclo.
- **Entregável:** custo da calda base calculado uma vez e reusado nos sabores.

### Fase 5 — Produção com lote/validade
- [ ] Tela **registrar produção** de um item (calda base ou sabor); nº de lotes.
- [ ] Baixa de componentes (FEFO) + geração de estoque do item com **lote e
      validade** + custo do lote, em transação (doc 02, 6.2). Bloquear se faltar insumo.
- **Entregável:** produzir baixa os componentes certos e credita estoque com lote.

### Fase 6 — Vendas, formatos e recibo (DOR #3, #6)
- [ ] Cadastro de **formatos de venda** por produto: bola, casquinha, cascão, pote,
      picolé, balde, **self-service por kg**, com preço **varejo e atacado**.
- [ ] **Produtos montados na venda** (`MONTADO_NA_VENDA`): sundae, milkshake, açaí
      com complementos — definir a composição (reusa ficha técnica) que baixa vários
      insumos/produtos na venda (doc 02, 6.4).
- [ ] **PDV**: escolher produto + formato; venda por unidade, **por peso (kg)** ou
      montado; canal varejo/atacado; forma de pagamento.
- [ ] Finalizar: baixa de estoque (FEFO) + `Venda`/`ItemVenda` (congela custo) +
      **recibo numerado por empresa** imprimível/PDF (doc 02, 6.3/6.4 e 8).
- [ ] **Ponto de integração de balança** (peso→preço): começar com peso digitado no
      PDV, deixar a interface pronta para leitura automática depois.
- **Entregável:** vender em qualquer formato (inclusive montado) gera recibo e baixa estoque.

### Fase 7 — Painel, CMV, margem e alertas (DOR #6)
- [ ] Dashboard: estoque de insumo/produto; **alertas de estoque mínimo** e de
      **validade próxima**.
- [ ] **CMV** (Custo da Mercadoria Vendida) por período como métrica de primeira classe.
- [ ] Relatório **custo x preço praticado x margem** por produto/formato
      (varejo e atacado).
- [ ] **Previsão de ruptura + sugestão de compra**: a partir do consumo médio,
      estimar "acaba em X dias" e sugerir reposição (ponto de pedido). Simples, sem IA.
- [ ] Relatórios de compras e de vendas por período.
- **Entregável:** o dono enxerga CMV, margem e o que precisa comprar/produzir.

### Fase 8+ — Futuro (fora do MVP)
- **PDV offline-first (PWA com fila de vendas)** — repriorizado: vender não pode
  parar por falta de internet (lição GranMoney/Saipos — ver doc 04).
- NFC-e/NF-e de saída (já nas regras da **Reforma Tributária 2026**); SPED; contador.
- Hardware: impressora térmica, **balança**, gaveta, maquininha/TEF.
- Rastreabilidade completa de lote / recall.
- Financeiro (contas a pagar/receber, fluxo de caixa, DRE).
- **Curva ABC de insumos**; previsão de demanda/sazonalidade; promoções.
- **Ordem de produção sugerida pela demanda/estoque**.
- **Pedido loja→fábrica (B2B interno)** para atacado/rede.
- Mobile; delivery / cardápio digital / iFood.
- Endurecimento multi-tenant com **RLS no PostgreSQL**.

## 5. Nota sobre OCR da foto da nota

O **XML** é a via preferida (dados estruturados e exatos — toda NF-e tem XML). A
**foto** cobre casos sem XML (cupom, nota de balcão) e é inerentemente menos
confiável, então **sempre passa por revisão humana** antes de virar compra.

Opções para extrair os itens da imagem (decidir na Fase 3):
- **Modelo de visão via API (ex.: Claude)** — enviar a imagem e pedir os itens em
  JSON estruturado (descrição, unidade, quantidade, valor unitário, total). Bom
  equilíbrio de qualidade e esforço.
- OCR tradicional (Tesseract) + parsing — mais trabalhoso e frágil com layouts variados.

Requisitos: guardar a imagem original (`arquivoUrl`), mostrar lado a lado
imagem × itens extraídos para conferência, e reaproveitar o mesmo mapeamento
produto-fornecedor do fluxo de XML.

## 6. Ambiente de desenvolvimento (notas)

- PostgreSQL 16 local (porta 5432):
  `DATABASE_URL="postgresql://postgres:postgres@localhost:5432/sorveteria"`.
- Node 22 / npm 10.
- Chromium já instalado (`PLAYWRIGHT_BROWSERS_PATH=/opt/pw-browsers`) — **não** rodar
  `playwright install`.
- **Não commitar** `.env`; fornecer `.env.example`.

## 7. Definição de pronto (Definition of Done) por fase

- `next build` passa sem erros de tipo.
- Teste do fluxo principal da fase (unit e/ou e2e).
- Todas as queries novas filtram por `empresaId`.
- Operações de estoque/custo/produção/venda em transação.
- UI em pt-BR e valores formatados corretamente.
