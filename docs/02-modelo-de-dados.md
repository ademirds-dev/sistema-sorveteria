# 02 — Modelo de Dados e Multi-Tenancy

> Documento de planejamento. O schema Prisma abaixo é uma **proposta de referência**
> para a execução, não o arquivo final. Ajustar conforme as decisões em aberto do doc 01.

## 1. Estratégia de multi-tenancy

**Escolha para o MVP: banco único, schema único, isolamento por coluna `empresaId`
(shared database, shared schema, row-level scoping).**

Por quê:

- É a abordagem mais simples e barata para começar um SaaS.
- Migrações rodam uma vez para todos os tenants.
- Escala bem para dezenas/centenas de sorveterias.

Regras inegociáveis para que o isolamento seja seguro:

1. **Toda tabela de negócio tem `empresaId`** (FK para `Empresa`).
2. **Nenhuma query cruza tenants.** Toda leitura/escrita filtra por `empresaId`.
3. O `empresaId` vem **sempre da sessão do usuário autenticado**, nunca de
   parâmetro enviado pelo cliente.
4. Centralizar o acesso a dados: usar uma **Prisma Client Extension** (ou helper
   `getTenantDb(empresaId)`) que injeta `where: { empresaId }` automaticamente,
   para evitar esquecer o filtro em alguma query.
5. **Unicidade é sempre composta com `empresaId`** (ex.: nome de insumo único
   _por empresa_, não global): `@@unique([empresaId, nome])`.

Endurecimento futuro (fase posterior, não no MVP): ativar **Row-Level Security
(RLS) no PostgreSQL** com `SET app.current_tenant`, para defesa em profundidade
no nível do banco. Deixar o código preparado (sempre passar `empresaId`) já
facilita essa migração.

## 2. Regras de tipos e precisão

- **Dinheiro e quantidades:** `Decimal` no Prisma (`@db.Decimal(14,4)` para
  quantidades/custos internos; exibir arredondado). **Nunca `Float`.**
- **IDs:** `cuid()` (string) — bom para sistemas distribuídos e não expõe volume.
- **Datas:** `DateTime` com `@default(now())` em `createdAt` e `@updatedAt`.
- **Enums** para papéis, tipos de movimento e método de custeio.
- **Soft delete** onde faz sentido em cadastros (`ativo Boolean @default(true)`)
  em vez de apagar registros com histórico.

## 3. Entidades principais

Resumo antes do schema:

- `Empresa` — o tenant.
- `Usuario` — pertence a uma empresa, tem papel.
- `Fornecedor` — de quem se compra.
- `Insumo` — matéria-prima/embalagem, com unidades e estoque.
- `CompraInsumo` + `ItemCompra` — entrada de estoque e base do custo.
- `MovimentoEstoqueInsumo` — razão (ledger) de toda entrada/saída de insumo.
- `Produto` — o que se fabrica/vende (tipo + sabor).
- `FichaTecnica` + `FichaTecnicaItem` — receita (insumos por lote + rendimento).
- `Producao` — lote fabricado; consome insumos, gera produto acabado.
- `MovimentoEstoqueProduto` — razão de entrada/saída de produto acabado.
- `Venda` + `ItemVenda` — saída e base do recibo.
- `Recibo` — comprovante numerado por empresa.

## 4. Schema Prisma de referência

```prisma
// datasource + generator ficam no schema.prisma real
// datasource db { provider = "postgresql"; url = env("DATABASE_URL") }
// generator client { provider = "prisma-client-js" }

enum Papel {
  OWNER
  GERENTE
  PRODUCAO
  CAIXA
}

enum MetodoCusteio {
  ULTIMA_COMPRA
  MEDIA_PONDERADA
}

enum TipoMovimento {
  ENTRADA
  SAIDA
  AJUSTE
}

model Empresa {
  id             String        @id @default(cuid())
  nome           String
  cnpj           String?
  telefone       String?
  endereco       String?
  metodoCusteio  MetodoCusteio @default(MEDIA_PONDERADA)
  ativo          Boolean       @default(true)
  createdAt      DateTime      @default(now())
  updatedAt      DateTime      @updatedAt

  usuarios       Usuario[]
  fornecedores   Fornecedor[]
  insumos        Insumo[]
  produtos       Produto[]
  compras        CompraInsumo[]
  producoes      Producao[]
  vendas         Venda[]
}

model Usuario {
  id           String   @id @default(cuid())
  empresaId    String
  empresa      Empresa  @relation(fields: [empresaId], references: [id])
  nome         String
  email        String
  senhaHash    String
  papel        Papel    @default(CAIXA)
  ativo        Boolean  @default(true)
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  @@unique([empresaId, email])
  @@index([empresaId])
}

model Fornecedor {
  id         String   @id @default(cuid())
  empresaId  String
  empresa    Empresa  @relation(fields: [empresaId], references: [id])
  nome       String
  telefone   String?
  cnpj       String?
  ativo      Boolean  @default(true)
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt

  compras    CompraInsumo[]

  @@unique([empresaId, nome])
  @@index([empresaId])
}

model Insumo {
  id              String   @id @default(cuid())
  empresaId       String
  empresa         Empresa  @relation(fields: [empresaId], references: [id])
  nome            String
  // Ex.: compra em "SACO" e usa em "G". fatorConversao = 25000 (1 saco = 25000 g)
  unidadeCompra   String
  unidadeUso      String
  fatorConversao  Decimal  @db.Decimal(14, 4) // quantas unidadeUso há em 1 unidadeCompra
  // Estoque sempre armazenado na unidade de USO (menor granularidade).
  estoqueAtual    Decimal  @default(0) @db.Decimal(14, 4)
  estoqueMinimo   Decimal  @default(0) @db.Decimal(14, 4)
  // Custos armazenados por unidade de USO.
  custoUltimo     Decimal  @default(0) @db.Decimal(14, 6)
  custoMedio      Decimal  @default(0) @db.Decimal(14, 6)
  ehEmbalagem     Boolean  @default(false)
  ativo           Boolean  @default(true)
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  itensCompra     ItemCompra[]
  fichaItens      FichaTecnicaItem[]
  movimentos      MovimentoEstoqueInsumo[]

  @@unique([empresaId, nome])
  @@index([empresaId])
}

model CompraInsumo {
  id            String     @id @default(cuid())
  empresaId     String
  empresa       Empresa    @relation(fields: [empresaId], references: [id])
  fornecedorId  String?
  fornecedor    Fornecedor? @relation(fields: [fornecedorId], references: [id])
  numeroNota    String?
  data          DateTime   @default(now())
  valorTotal    Decimal    @db.Decimal(14, 2)
  createdAt     DateTime   @default(now())
  criadoPorId   String?

  itens         ItemCompra[]

  @@index([empresaId])
  @@index([empresaId, data])
}

model ItemCompra {
  id              String   @id @default(cuid())
  empresaId       String
  compraId        String
  compra          CompraInsumo @relation(fields: [compraId], references: [id])
  insumoId        String
  insumo          Insumo   @relation(fields: [insumoId], references: [id])
  // Quantidade na unidade de COMPRA (ex.: 2 sacos)
  quantidadeCompra Decimal @db.Decimal(14, 4)
  // Valor total pago por este item
  valorItem        Decimal @db.Decimal(14, 2)

  @@index([empresaId])
  @@index([compraId])
}

// Razão (ledger) de estoque de insumo: fonte de verdade do histórico.
model MovimentoEstoqueInsumo {
  id           String        @id @default(cuid())
  empresaId    String
  insumoId     String
  insumo       Insumo        @relation(fields: [insumoId], references: [id])
  tipo         TipoMovimento
  // Quantidade na unidade de USO, sempre positiva; o sinal vem do tipo.
  quantidade   Decimal       @db.Decimal(14, 4)
  custoUnitario Decimal      @db.Decimal(14, 6) // custo por unidade de uso no momento
  origem       String        // "COMPRA", "PRODUCAO", "AJUSTE"
  origemId     String?       // id da compra/produção que gerou o movimento
  data         DateTime      @default(now())

  @@index([empresaId])
  @@index([insumoId])
}

model Produto {
  id             String   @id @default(cuid())
  empresaId      String
  empresa        Empresa  @relation(fields: [empresaId], references: [id])
  nome           String   // ex.: "Sorvete de Chocolate 2L"
  tipo           String   // ex.: "MASSA", "PICOLE", "ACAI", "SOFT", "CASQUINHA"
  sabor          String?
  // Unidade de venda do produto acabado (ex.: "POTE_2L", "BALDE_10L", "UNIDADE")
  unidadeVenda   String
  precoVenda     Decimal  @default(0) @db.Decimal(14, 2)
  estoqueAtual   Decimal  @default(0) @db.Decimal(14, 4)
  ativo          Boolean  @default(true)
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt

  fichas         FichaTecnica[]
  producoes      Producao[]
  itensVenda     ItemVenda[]
  movimentos     MovimentoEstoqueProduto[]

  @@unique([empresaId, nome])
  @@index([empresaId])
}

model FichaTecnica {
  id            String   @id @default(cuid())
  empresaId     String
  produtoId     String
  produto       Produto  @relation(fields: [produtoId], references: [id])
  // Quanto um lote rende, na unidade de venda do produto (ex.: rende 5 potes)
  rendimento    Decimal  @db.Decimal(14, 4)
  ativa         Boolean  @default(true)
  versao        Int      @default(1)
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  itens         FichaTecnicaItem[]

  @@index([empresaId])
  @@index([produtoId])
}

model FichaTecnicaItem {
  id              String   @id @default(cuid())
  empresaId       String
  fichaId         String
  ficha           FichaTecnica @relation(fields: [fichaId], references: [id])
  insumoId        String
  insumo          Insumo   @relation(fields: [insumoId], references: [id])
  // Quantidade de insumo (unidade de USO) para produzir UM lote inteiro.
  quantidade      Decimal  @db.Decimal(14, 4)

  @@index([empresaId])
  @@index([fichaId])
}

model Producao {
  id            String   @id @default(cuid())
  empresaId     String
  empresa       Empresa  @relation(fields: [empresaId], references: [id])
  produtoId     String
  produto       Produto  @relation(fields: [produtoId], references: [id])
  fichaId       String?
  // Quantos lotes foram produzidos (multiplica as quantidades da ficha)
  qtdLotes      Decimal  @default(1) @db.Decimal(14, 4)
  // Quantidade de produto acabado gerada (rendimento * qtdLotes)
  qtdProduzida  Decimal  @db.Decimal(14, 4)
  custoTotal    Decimal  @db.Decimal(14, 2) // custo dos insumos consumidos
  custoUnitario Decimal  @db.Decimal(14, 6) // custoTotal / qtdProduzida
  numeroLote    String?
  validade      DateTime?
  data          DateTime @default(now())
  criadoPorId   String?

  @@index([empresaId])
  @@index([empresaId, data])
}

model MovimentoEstoqueProduto {
  id           String        @id @default(cuid())
  empresaId    String
  produtoId    String
  produto      Produto       @relation(fields: [produtoId], references: [id])
  tipo         TipoMovimento
  quantidade   Decimal       @db.Decimal(14, 4)
  custoUnitario Decimal      @db.Decimal(14, 6)
  origem       String        // "PRODUCAO", "VENDA", "AJUSTE"
  origemId     String?
  data         DateTime      @default(now())

  @@index([empresaId])
  @@index([produtoId])
}

model Venda {
  id           String   @id @default(cuid())
  empresaId    String
  empresa      Empresa  @relation(fields: [empresaId], references: [id])
  data         DateTime @default(now())
  valorTotal   Decimal  @db.Decimal(14, 2)
  formaPagamento String? // "DINHEIRO", "PIX", "CARTAO"
  criadoPorId  String?
  createdAt    DateTime @default(now())

  itens        ItemVenda[]
  recibo       Recibo?

  @@index([empresaId])
  @@index([empresaId, data])
}

model ItemVenda {
  id            String   @id @default(cuid())
  empresaId     String
  vendaId       String
  venda         Venda    @relation(fields: [vendaId], references: [id])
  produtoId     String
  produto       Produto  @relation(fields: [produtoId], references: [id])
  quantidade    Decimal  @db.Decimal(14, 4)
  precoUnitario Decimal  @db.Decimal(14, 2)
  custoUnitario Decimal  @db.Decimal(14, 6) // congelado no momento da venda (margem)

  @@index([empresaId])
  @@index([vendaId])
}

model Recibo {
  id         String   @id @default(cuid())
  empresaId  String
  vendaId    String   @unique
  venda      Venda    @relation(fields: [vendaId], references: [id])
  // Numeração sequencial POR EMPRESA (ver seção 6)
  numero     Int
  emitidoEm  DateTime @default(now())

  @@unique([empresaId, numero])
  @@index([empresaId])
}
```

## 5. Regras de negócio de custo e estoque

Estas regras são o núcleo do valor do sistema. A execução deve implementá-las
**dentro de transações** (`prisma.$transaction`) para nunca deixar estoque e
custo inconsistentes.

### 5.1 Ao registrar uma compra de insumo

Para cada item da compra:

1. Converter a quantidade comprada para a unidade de uso:
   `qtdUso = quantidadeCompra * fatorConversao`.
2. Custo unitário desta compra (por unidade de uso):
   `custoCompra = valorItem / qtdUso`.
3. Atualizar `custoUltimo = custoCompra`.
4. Atualizar `custoMedio` (média ponderada móvel):
   ```
   custoMedio = (estoqueAtual * custoMedio + qtdUso * custoCompra)
                / (estoqueAtual + qtdUso)
   ```
5. `estoqueAtual += qtdUso`.
6. Criar `MovimentoEstoqueInsumo` do tipo `ENTRADA` com `origem = "COMPRA"`.

O **custo usado no cálculo do produto** depende de `empresa.metodoCusteio`
(`ULTIMA_COMPRA` → `custoUltimo`; `MEDIA_PONDERADA` → `custoMedio`).

### 5.2 Custo de um produto (via ficha técnica)

`custoProduto = Σ (item.quantidade * custoDoInsumo) / ficha.rendimento`

onde `custoDoInsumo` respeita o método de custeio da empresa. Este valor é
calculado sob demanda (não precisa ser persistido; pode ser cacheado depois).

### 5.3 Ao registrar produção

Dentro de uma transação:

1. Para cada item da ficha: `consumo = item.quantidade * qtdLotes`.
2. Validar estoque suficiente de cada insumo (ou permitir negativo com alerta —
   decisão de UX; recomendo bloquear e avisar).
3. Baixar estoque de cada insumo e criar `MovimentoEstoqueInsumo` `SAIDA`
   (`origem = "PRODUCAO"`).
4. `custoTotal = Σ (consumo * custoDoInsumo)`.
5. `qtdProduzida = ficha.rendimento * qtdLotes`;
   `custoUnitario = custoTotal / qtdProduzida`.
6. Aumentar `produto.estoqueAtual += qtdProduzida` e criar
   `MovimentoEstoqueProduto` `ENTRADA` (`origem = "PRODUCAO"`).

### 5.4 Ao registrar venda

Dentro de uma transação:

1. Para cada item: validar estoque de produto acabado.
2. Baixar `produto.estoqueAtual` e criar `MovimentoEstoqueProduto` `SAIDA`
   (`origem = "VENDA"`), congelando `custoUnitario` do produto no momento (para
   cálculo de margem histórica correta).
3. Somar `valorTotal`.
4. Gerar `Recibo` com número sequencial da empresa.

## 6. Numeração de recibo por empresa

Não usar `count()+1` (condição de corrida). Opções:

- Uma tabela/contador por empresa atualizada dentro da mesma transação da venda; ou
- `SELECT ... FOR UPDATE` no contador; ou
- Sequência dedicada.

Garantir unicidade com `@@unique([empresaId, numero])` como rede de segurança.

## 7. Índices e performance

- Todo modelo de negócio tem `@@index([empresaId])`.
- Consultas por período usam índice composto `([empresaId, data])`.
- Os modelos `Movimento*` são o **ledger**: nunca editar/apagar; corrigir com
  lançamento de `AJUSTE`. Eles permitem reconstruir estoque e auditar.
