# 02 — Modelo de Dados e Multi-Tenancy

> Documento de planejamento. O schema Prisma abaixo é uma **proposta de referência**
> para a execução, não o arquivo final. Ajustar conforme as decisões em aberto do doc 01.

## 1. Estratégia de multi-tenancy

**Escolha para o MVP: banco único, schema único, isolamento por coluna `empresaId`
(shared database, shared schema, row-level scoping).**

Regras inegociáveis:

1. **Toda tabela de negócio tem `empresaId`** (FK para `Empresa`).
2. **Nenhuma query cruza tenants.** Toda leitura/escrita filtra por `empresaId`.
3. O `empresaId` vem **sempre da sessão** do usuário autenticado, nunca do cliente.
4. Centralizar o acesso: **Prisma Client Extension** / helper `getTenantDb(empresaId)`
   que injeta `where: { empresaId }` automaticamente.
5. **Unicidade sempre composta com `empresaId`** (ex.: `@@unique([empresaId, nome])`).

Endurecimento futuro (fora do MVP): **Row-Level Security (RLS)** no PostgreSQL.

## 2. Regras de tipos e precisão

- **Dinheiro e quantidades:** `Decimal` (`@db.Decimal(14,4)` para quantidades/custos;
  `@db.Decimal(14,2)` para valores monetários exibidos). **Nunca `Float`.**
- **IDs:** `cuid()`.
- **Datas:** `DateTime`, com `createdAt`/`updatedAt`.
- **Enums** para papéis, tipos de item, tipos de movimento, método de custeio,
  status de importação de NF.
- **Soft delete** em cadastros (`ativo Boolean @default(true)`).

## 3. Decisão-chave: modelo unificado `Item` + ficha técnica multi-nível

Para atender **custo da calda base + custo do produto acabado**, o sistema modela
**insumo, semi-acabado e produto acabado como um único tipo `Item`** (diferenciados
por `tipo`), e a ficha técnica (`Receita`) referencia **outros itens** como
componentes. Isso dá uma **lista de materiais (BOM) em vários níveis**:

```
Sorvete de Chocolate (PRODUTO)
├── Calda Base (INTERMEDIÁRIO)      ← custo calculado uma vez, reusado por todos
│   ├── Leite (INSUMO)
│   ├── Açúcar (INSUMO)
│   ├── Estabilizante (INSUMO)
│   └── Emulsificante (INSUMO)
├── Chocolate em pó (INSUMO)
└── Pote 2L + tampa (INSUMO/embalagem)
```

O custo sobe em cascata: calcula-se o custo da calda base a partir dos insumos e,
em seguida, o custo do sabor a partir da calda + saborizante + embalagem.

## 4. Schema Prisma de referência

```prisma
// datasource db { provider = "postgresql"; url = env("DATABASE_URL") }
// generator client { provider = "prisma-client-js" }

enum Papel        { OWNER GERENTE PRODUCAO CAIXA }
enum MetodoCusteio { ULTIMA_COMPRA MEDIA_PONDERADA }
enum TipoItem     { INSUMO INTERMEDIARIO PRODUTO }
enum TipoMovimento { ENTRADA SAIDA AJUSTE }
enum StatusNota   { PENDENTE_MAPEAMENTO CONFIRMADA DESCARTADA }
enum OrigemNota   { XML FOTO }

model Empresa {
  id            String        @id @default(cuid())
  nome          String
  cnpj          String?
  telefone      String?
  endereco      String?
  metodoCusteio MetodoCusteio @default(MEDIA_PONDERADA)
  ativo         Boolean       @default(true)
  createdAt     DateTime      @default(now())
  updatedAt     DateTime      @updatedAt

  usuarios      Usuario[]
  fornecedores  Fornecedor[]
  itens         Item[]
  compras       CompraInsumo[]
  producoes     Producao[]
  vendas        Venda[]
  notas         NotaFiscalImportada[]
}

model Usuario {
  id        String   @id @default(cuid())
  empresaId String
  empresa   Empresa  @relation(fields: [empresaId], references: [id])
  nome      String
  email     String
  senhaHash String
  papel     Papel    @default(CAIXA)
  ativo     Boolean  @default(true)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@unique([empresaId, email])
  @@index([empresaId])
}

model Fornecedor {
  id        String   @id @default(cuid())
  empresaId String
  empresa   Empresa  @relation(fields: [empresaId], references: [id])
  nome      String
  telefone  String?
  cnpj      String?
  ativo     Boolean  @default(true)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  compras      CompraInsumo[]
  mapeamentos  MapeamentoInsumoFornecedor[]
  notas        NotaFiscalImportada[]

  @@unique([empresaId, nome])
  @@index([empresaId])
}

// Modelo unificado: INSUMO, INTERMEDIARIO (calda base) e PRODUTO.
model Item {
  id             String   @id @default(cuid())
  empresaId      String
  empresa        Empresa  @relation(fields: [empresaId], references: [id])
  nome           String
  tipo           TipoItem
  // Unidade em que o ESTOQUE é controlado (menor granularidade): G, ML, KG, L, UN.
  unidadeEstoque String
  // Só para INSUMO comprado em outra unidade (ex.: SACO, CX, L):
  unidadeCompra  String?
  fatorConversao Decimal? @db.Decimal(14, 4) // quantas unidadeEstoque há em 1 unidadeCompra
  estoqueAtual   Decimal  @default(0) @db.Decimal(14, 4) // na unidadeEstoque
  estoqueMinimo  Decimal  @default(0) @db.Decimal(14, 4)
  // Custos por unidade de ESTOQUE (para INSUMO vêm das compras; para produzidos,
  // podem ser recalculados/cacheados a partir da última produção).
  custoUltimo    Decimal  @default(0) @db.Decimal(14, 6)
  custoMedio     Decimal  @default(0) @db.Decimal(14, 6)
  ehEmbalagem    Boolean  @default(false)
  controlaLote   Boolean  @default(false)
  ativo          Boolean  @default(true)
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt

  receita        Receita?           // ficha técnica que PRODUZ este item (INTERMEDIARIO/PRODUTO)
  usadoEmItens   ComponenteReceita[] @relation("ComponenteItem") // onde é componente
  itensCompra    ItemCompra[]
  movimentos     MovimentoEstoque[]
  lotes          Lote[]
  formatosVenda  FormatoVenda[]
  itensVenda     ItemVenda[]
  producoes      Producao[]
  itensNota      NotaFiscalItemImportado[]
  mapeamentos    MapeamentoInsumoFornecedor[]

  @@unique([empresaId, nome])
  @@index([empresaId, tipo])
}

// Ficha técnica: receita que produz UM item (calda base ou sabor).
model Receita {
  id          String   @id @default(cuid())
  empresaId   String
  itemId      String   @unique
  item        Item     @relation(fields: [itemId], references: [id])
  // Quanto um lote rende, na unidadeEstoque do item produzido (ex.: 10 L de calda).
  rendimento  Decimal  @db.Decimal(14, 4)
  versao      Int      @default(1)
  ativa       Boolean  @default(true)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  componentes ComponenteReceita[]

  @@index([empresaId])
}

// Um componente da receita: OUTRO item (insumo ou intermediário) + quantidade.
model ComponenteReceita {
  id               String  @id @default(cuid())
  empresaId        String
  receitaId        String
  receita          Receita @relation(fields: [receitaId], references: [id])
  componenteItemId String
  componenteItem   Item    @relation("ComponenteItem", fields: [componenteItemId], references: [id])
  // Quantidade do componente (na unidadeEstoque dele) para produzir UM lote inteiro.
  quantidade       Decimal @db.Decimal(14, 4)

  @@index([empresaId])
  @@index([receitaId])
}

model CompraInsumo {
  id           String      @id @default(cuid())
  empresaId    String
  empresa      Empresa     @relation(fields: [empresaId], references: [id])
  fornecedorId String?
  fornecedor   Fornecedor? @relation(fields: [fornecedorId], references: [id])
  numeroNota   String?
  data         DateTime    @default(now())
  valorTotal   Decimal     @db.Decimal(14, 2)
  // Se veio de importação de NF:
  notaId       String?     @unique
  createdAt    DateTime    @default(now())
  criadoPorId  String?

  itens        ItemCompra[]

  @@index([empresaId])
  @@index([empresaId, data])
}

model ItemCompra {
  id               String       @id @default(cuid())
  empresaId        String
  compraId         String
  compra           CompraInsumo @relation(fields: [compraId], references: [id])
  itemId           String       // sempre um Item do tipo INSUMO
  item             Item         @relation(fields: [itemId], references: [id])
  quantidadeCompra Decimal      @db.Decimal(14, 4) // na unidadeCompra
  valorItem        Decimal      @db.Decimal(14, 2)
  loteId           String?      // se o insumo controla lote/validade

  @@index([empresaId])
  @@index([compraId])
}

// Lote de estoque (insumo perecível ou produto fabricado).
model Lote {
  id                String   @id @default(cuid())
  empresaId         String
  itemId            String
  item              Item     @relation(fields: [itemId], references: [id])
  codigo            String
  validade          DateTime?
  quantidadeInicial Decimal  @db.Decimal(14, 4)
  quantidadeAtual   Decimal  @db.Decimal(14, 4)
  custoUnitario     Decimal  @db.Decimal(14, 6)
  producaoId        String?
  createdAt         DateTime @default(now())

  @@unique([empresaId, itemId, codigo])
  @@index([empresaId, itemId])
  @@index([empresaId, validade]) // para FEFO e alerta de validade
}

// Razão (ledger) único de estoque, para qualquer Item.
model MovimentoEstoque {
  id            String        @id @default(cuid())
  empresaId     String
  itemId        String
  item          Item          @relation(fields: [itemId], references: [id])
  loteId        String?
  tipo          TipoMovimento
  quantidade    Decimal       @db.Decimal(14, 4) // na unidadeEstoque, sempre positiva
  custoUnitario Decimal       @db.Decimal(14, 6)
  origem        String        // "COMPRA" | "PRODUCAO_CONSUMO" | "PRODUCAO_ENTRADA" | "VENDA" | "AJUSTE"
  origemId      String?
  data          DateTime      @default(now())

  @@index([empresaId])
  @@index([empresaId, itemId])
}

// Produção de um item (calda base OU sabor).
model Producao {
  id            String    @id @default(cuid())
  empresaId     String
  empresa       Empresa   @relation(fields: [empresaId], references: [id])
  itemId        String    // item produzido
  item          Item      @relation(fields: [itemId], references: [id])
  receitaId     String?
  qtdLotes      Decimal   @default(1) @db.Decimal(14, 4) // multiplica as quantidades da receita
  qtdProduzida  Decimal   @db.Decimal(14, 4)             // rendimento * qtdLotes
  custoTotal    Decimal   @db.Decimal(14, 2)
  custoUnitario Decimal   @db.Decimal(14, 6)
  numeroLote    String?
  validade      DateTime?
  data          DateTime  @default(now())
  criadoPorId   String?

  @@index([empresaId])
  @@index([empresaId, data])
}

// Formato/embalagem de venda de um produto acabado.
model FormatoVenda {
  id                String   @id @default(cuid())
  empresaId         String
  itemId            String   // Item do tipo PRODUTO
  item              Item     @relation(fields: [itemId], references: [id])
  nome              String   // "Bola 100g", "Casquinha", "Cascão", "Pote 2L", "Picolé", "Balde 10L", "Self-service (kg)"
  // Quanto do estoque (unidadeEstoque do item) sai por 1 unidade vendida.
  // Para venda por peso (self-service/kg), fica null e porPeso = true.
  quantidadeEstoque Decimal? @db.Decimal(14, 4)
  porPeso           Boolean  @default(false)
  precoVarejo       Decimal  @db.Decimal(14, 2)
  precoAtacado      Decimal? @db.Decimal(14, 2)
  ativo             Boolean  @default(true)

  itensVenda        ItemVenda[]

  @@unique([empresaId, itemId, nome])
  @@index([empresaId])
}

model Venda {
  id             String   @id @default(cuid())
  empresaId      String
  empresa        Empresa  @relation(fields: [empresaId], references: [id])
  data           DateTime @default(now())
  canal          String   @default("VAREJO") // "VAREJO" | "ATACADO"
  valorTotal     Decimal  @db.Decimal(14, 2)
  formaPagamento String?  // "DINHEIRO" | "PIX" | "CARTAO"
  criadoPorId    String?
  createdAt      DateTime @default(now())

  itens          ItemVenda[]
  recibo         Recibo?

  @@index([empresaId])
  @@index([empresaId, data])
}

model ItemVenda {
  id             String       @id @default(cuid())
  empresaId      String
  vendaId        String
  venda          Venda        @relation(fields: [vendaId], references: [id])
  itemId         String
  item           Item         @relation(fields: [itemId], references: [id])
  formatoVendaId String?
  formatoVenda   FormatoVenda? @relation(fields: [formatoVendaId], references: [id])
  // Quantidade vendida na unidade do formato (nº de bolas, kg no self-service, unidades).
  quantidade     Decimal      @db.Decimal(14, 4)
  // Quantidade baixada do estoque (na unidadeEstoque do item) — resolve fração/kg.
  quantidadeEstoque Decimal   @db.Decimal(14, 4)
  precoUnitario  Decimal      @db.Decimal(14, 2)
  custoUnitario  Decimal      @db.Decimal(14, 6) // congelado no momento (margem histórica)

  @@index([empresaId])
  @@index([vendaId])
}

model Recibo {
  id        String   @id @default(cuid())
  empresaId String
  vendaId   String   @unique
  venda     Venda    @relation(fields: [vendaId], references: [id])
  numero    Int      // sequencial POR EMPRESA (ver seção 6)
  emitidoEm DateTime @default(now())

  @@unique([empresaId, numero])
  @@index([empresaId])
}

// ---------- Importação de Nota Fiscal ----------

model NotaFiscalImportada {
  id           String      @id @default(cuid())
  empresaId    String
  empresa      Empresa     @relation(fields: [empresaId], references: [id])
  fornecedorId String?
  fornecedor   Fornecedor? @relation(fields: [fornecedorId], references: [id])
  origem       OrigemNota
  status       StatusNota  @default(PENDENTE_MAPEAMENTO)
  chaveNfe     String?     // 44 dígitos (XML)
  numero       String?
  dataEmissao  DateTime?
  valorTotal   Decimal?    @db.Decimal(14, 2)
  rawXml       String?     @db.Text // XML original (origem XML)
  arquivoUrl   String?     // imagem original (origem FOTO)
  compraId     String?     // compra gerada ao confirmar
  createdAt    DateTime    @default(now())

  itens        NotaFiscalItemImportado[]

  @@index([empresaId])
  @@index([empresaId, status])
}

model NotaFiscalItemImportado {
  id                  String   @id @default(cuid())
  empresaId           String
  notaId              String
  nota                NotaFiscalImportada @relation(fields: [notaId], references: [id])
  // Dados como vieram da NF/foto:
  descricaoFornecedor String
  codigoFornecedor    String?  // cProd (XML)
  ean                 String?  // cEAN (XML)
  ncm                 String?
  unidade             String?  // uCom
  quantidade          Decimal? @db.Decimal(14, 4)
  valorUnitario       Decimal? @db.Decimal(14, 6)
  valorTotal          Decimal? @db.Decimal(14, 2)
  // Mapeamento resolvido:
  itemId              String?  // Item (INSUMO) correspondente
  item                Item?    @relation(fields: [itemId], references: [id])

  @@index([empresaId])
  @@index([notaId])
}

// "Memória" do mapeamento produto-do-fornecedor -> nosso insumo.
model MapeamentoInsumoFornecedor {
  id               String     @id @default(cuid())
  empresaId        String
  fornecedorId     String
  fornecedor       Fornecedor @relation(fields: [fornecedorId], references: [id])
  codigoFornecedor String?    // cProd
  ean              String?
  itemId           String     // Item (INSUMO)
  item             Item       @relation(fields: [itemId], references: [id])
  // Converte a quantidade da NF (unidade do fornecedor) para a unidadeEstoque do insumo.
  fatorConversao   Decimal    @db.Decimal(14, 4)
  createdAt        DateTime   @default(now())

  @@unique([empresaId, fornecedorId, codigoFornecedor])
  @@index([empresaId])
}
```

## 5. Regras de custo em cascata (calda base + produto)

O custo é calculado de baixo para cima (roll-up). Implementar como função pura em
`lib/custeio.ts`, com memoização para não recalcular a mesma calda várias vezes.

```
custoUnitario(item):
  se item.tipo == INSUMO:
    retorna (metodoCusteio == ULTIMA_COMPRA ? item.custoUltimo : item.custoMedio)
  senão (tem receita):
    total = Σ  componente.quantidade * custoUnitario(componente.item)
    retorna total / item.receita.rendimento
```

- **Custo da calda base** = soma dos insumos da calda ÷ rendimento da calda.
- **Custo do sabor** = (qtd_calda × custoCaldaBase + saborizante + embalagem) ÷ rendimento.
- Proteções: **detectar ciclo** na BOM (item que dependa de si mesmo) e barrar;
  respeitar `empresa.metodoCusteio`.

## 6. Regras de estoque (transações)

Todas dentro de `prisma.$transaction`. O `MovimentoEstoque` é o ledger; nunca editar/apagar.

### 6.1 Compra de insumo (manual ou confirmada de uma NF)
Para cada item:
1. `qtdEstoque = quantidadeCompra * fatorConversao`.
2. `custoCompra = valorItem / qtdEstoque`.
3. `custoUltimo = custoCompra`;
   `custoMedio = (estoqueAtual*custoMedio + qtdEstoque*custoCompra)/(estoqueAtual+qtdEstoque)`.
4. `estoqueAtual += qtdEstoque`.
5. Se `controlaLote`: criar/atualizar `Lote` (com validade informada).
6. `MovimentoEstoque` ENTRADA, `origem="COMPRA"`.

### 6.2 Produção (calda base ou sabor)
1. Para cada componente: `consumo = componente.quantidade * qtdLotes`.
2. Validar estoque (bloquear e avisar se faltar). Para componentes com lote, baixar
   por **FEFO** (vence primeiro sai primeiro).
3. Baixar estoque + `MovimentoEstoque` SAIDA (`origem="PRODUCAO_CONSUMO"`).
4. `custoTotal = Σ consumo * custoUnitario(componente)`.
5. `qtdProduzida = rendimento * qtdLotes`; `custoUnitario = custoTotal/qtdProduzida`.
6. Criar `Lote` do item produzido (código + validade) e dar ENTRADA
   (`origem="PRODUCAO_ENTRADA"`); atualizar `estoqueAtual` e custos do item.

### 6.3 Venda (qualquer formato)
1. Para cada item: resolver `quantidadeEstoque`:
   - formato por unidade: `quantidade * formato.quantidadeEstoque`;
   - formato por peso (self-service/kg): `quantidade` já é a qtd em unidadeEstoque.
2. Validar estoque; baixar por **FEFO** nos lotes; `MovimentoEstoque` SAIDA
   (`origem="VENDA"`), congelando `custoUnitario` do item.
3. `precoUnitario` conforme `canal` (varejo/atacado) do formato.
4. Somar `valorTotal`; gerar `Recibo` numerado.

## 7. Importação de NF (fluxo de dados)

**XML (NF-e):**
1. Upload do XML → parse dos itens (`det`: `cProd`, `xProd`, `cEAN`, `NCM`, `uCom`,
   `qCom`, `vUnCom`, `vProd`) e do emitente (vira/associa `Fornecedor`).
2. Cria `NotaFiscalImportada` (status `PENDENTE_MAPEAMENTO`) + `NotaFiscalItemImportado`.
3. Auto-mapeia via `MapeamentoInsumoFornecedor` (por `codigoFornecedor`/`ean`).
4. Usuário revisa itens não mapeados, escolhe/cria o `Item` (INSUMO) e o fator de
   conversão — **o mapeamento é salvo** para as próximas notas.
5. Ao **confirmar**: cria `CompraInsumo` + `ItemCompra` e roda a regra 6.1
   (entrada de estoque + custo). Status → `CONFIRMADA`.

**Foto (OCR):**
- Mesmo fluxo, mas os itens vêm de extração por OCR/visão (ver doc 03 para opções de
  serviço). **Revisão humana é obrigatória** antes de confirmar, pois a confiança é
  menor que a do XML.

## 8. Numeração de recibo por empresa

Não usar `count()+1` (condição de corrida). Usar contador por empresa atualizado na
mesma transação da venda (ou `SELECT ... FOR UPDATE` / sequência). Rede de
segurança: `@@unique([empresaId, numero])`.

## 9. Índices e performance

- Todo modelo tem `@@index([empresaId])`; consultas por período usam `([empresaId, data])`.
- `Lote` indexado por `validade` (FEFO + alertas de validade).
- `MovimentoEstoque` é o ledger: nunca editar/apagar; corrigir com `AJUSTE`.
