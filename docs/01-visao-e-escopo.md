# 01 — Visão, Público e Escopo

> Documento de planejamento. Nenhuma linha deste arquivo é código de execução —
> é a fonte de verdade sobre **o que** o sistema faz e **para quem**.

## 1. Visão

Sistema de gestão para **sorveteria que também é fábrica de sorvete** (produz o
próprio sorvete e vende no varejo e no atacado). O foco não é só o ponto de venda:
é conectar **compra de insumo → custo real → calda base → ficha técnica por sabor →
produção → estoque (com lote/validade) → venda (em vários formatos) → recibo →
margem de lucro**.

O sistema nasce para resolver dores concretas de uma sorveteria com ~40 anos de
operação, mas é construído desde o início como **SaaS multi-empresa (multi-tenant)**,
para no futuro ser vendido a outras sorveterias.

## 2. Dores que originaram o projeto (declaradas pelo cliente)

1. **Controle de estoque** de insumos, com **lote e validade**.
2. **Custo do produto sempre atualizado**, recalculado a partir das últimas
   compras — incluindo o **custo da calda base** (compartilhada entre sabores) e o
   **custo do produto acabado** (calda base + saborizante + embalagem).
3. **Emissão de recibo** de venda.
4. **Divisão de insumos por fabricação** — ficha técnica por sabor/tipo, com a
   calda base como componente comum.
5. **Entrada de estoque a partir da nota fiscal** — importar por **XML da NF-e** e
   por **foto da nota (OCR)**, mapeando os produtos automaticamente para evitar
   digitação e erro de cadastro.
6. **Preços praticados na venda e margem** — comparar preço x custo e mostrar a
   margem por produto/formato.

Essas dores são o **coração do MVP**.

## 3. Personas (quem usa)

| Persona | Papel no sistema | Precisa de |
|---|---|---|
| **Dono / Gerente** | administra tudo | cadastros, custos, importação de NF, relatórios, margem |
| **Produção / Fábrica** | fabrica calda base e sabores | fichas técnicas, registrar produção, ver estoque de insumo |
| **Caixa / Balconista** | vende no balcão | PDV (bola, casquinha, kg, pote...), gerar recibo |

Papéis de acesso (roles): `OWNER`, `GERENTE`, `PRODUCAO`, `CAIXA`. Cada empresa
(tenant) tem seus próprios usuários.

## 4. Conceitos do domínio (glossário)

- **Insumo (matéria-prima):** tudo que é comprado. Inclui ingredientes (leite,
  creme, açúcar, glucose, estabilizante, emulsificante, polpa, chocolate, castanha,
  essência) **e embalagens** (pote, tampa, colher, casquinha, cascão, saco, balde,
  palito). Embalagem também tem custo e entra no custo do produto.
- **Unidade de medida + conversão:** compra-se em uma unidade (saco de 25 kg, caixa,
  litro) e consome-se em outra (g, ml) na receita. **Conversão obrigatória.**
- **Item (modelo unificado):** para dar conta da calda base, o sistema trata
  **insumo, semi-acabado e produto acabado como um mesmo tipo de registro (`Item`),
  diferenciado por `tipo`**. Isso permite ficha técnica em vários níveis (ver doc 02).
  - **INSUMO** — comprado (matéria-prima / embalagem).
  - **INTERMEDIÁRIO (semi-acabado)** — produzido e usado em outros itens. Ex.:
    **calda base**, comum a vários sabores.
  - **PRODUTO** — produto acabado vendável. Ex.: sorvete de chocolate, picolé.
- **Calda base:** semi-acabado (INTERMEDIÁRIO) com ficha técnica e **custo próprio
  por litro/kg**. É componente de praticamente todos os sabores; calcular seu custo
  uma vez e reaproveitar é um requisito central.
- **Ficha técnica (receita):** lista de componentes (insumos e/ou intermediários) e
  quantidades para produzir um lote de um item, com **rendimento**. Como a calda
  base é um item, um sabor pode ter como componentes "X L de calda base + Y g de
  chocolate + 1 pote".
- **Produção (lote):** fabricar um item. Consome componentes (baixa estoque),
  gera o item produzido em estoque **com número de lote e validade**, e registra o
  custo do lote.
- **Formato de venda:** como um produto acabado é vendido. Ex.: **bola, casquinha,
  cascão, pote (2 L etc.), picolé, self-service por kg, balde (atacado)**. Cada
  formato tem um fator de baixa no estoque e **preço de varejo e/ou atacado**.
  Formatos por peso (self-service/kg) têm quantidade variável informada na venda.
- **Lote / validade:** rastreamento do estoque por lote, com data de validade,
  tanto de insumos perecíveis quanto de produtos fabricados.
- **Importação de NF:** entrada de compra a partir da nota fiscal, por **XML** (NF-e)
  ou **foto (OCR)**, com **mapeamento** do produto do fornecedor para o insumo
  cadastrado (memorizado para as próximas notas).
- **Recibo:** comprovante simples da venda (não é nota fiscal).

## 5. Escopo do MVP (versão 1)

Objetivo: **resolver as 6 dores declaradas, ponta a ponta, para uma empresa.**

Entra no MVP:

- [ ] **Empresa (tenant)** + **usuários** com papéis + login.
- [ ] **Fornecedores**.
- [ ] **Itens** (unificado): cadastro de INSUMO, INTERMEDIÁRIO e PRODUTO, com
      unidade de estoque/compra, fator de conversão, estoque mínimo, flag embalagem
      e flag de controle de lote.
- [ ] **Compra de insumo** manual que dá entrada no estoque e **atualiza o custo**
      (última compra + custo médio ponderado).
- [ ] **Importação de NF por XML (NF-e)**: lê os itens, mapeia por
      código/EAN → insumo (memoriza o mapeamento), gera a compra e a entrada de
      estoque após revisão/confirmação.
- [ ] **Importação de NF por foto (OCR)**: extrai os itens da imagem, mesmo fluxo
      de revisão/mapeamento/confirmação (com revisão humana obrigatória).
- [ ] **Ficha técnica multi-nível**: calda base (INTERMEDIÁRIO) e sabores (PRODUTO)
      compostos por insumos e/ou pela calda base, com rendimento.
- [ ] **Custo calculado em cascata**: custo da calda base e custo do produto acabado
      (calda + saborizante + embalagem), a partir do custo atual dos insumos.
- [ ] **Produção com lote e validade**: registrar produção de calda base e de
      sabores, baixando componentes e gerando estoque com lote/validade e custo do lote.
- [ ] **Formatos de venda** por produto: bola, casquinha, cascão, pote, picolé,
      balde, **self-service por kg**, com preço de **varejo e atacado**.
- [ ] **Produtos montados na venda** (sundae, milkshake, açaí com complementos):
      composição que baixa vários insumos/produtos no momento da venda.
- [ ] **CMV** e **previsão de ruptura/sugestão de compra** nos relatórios.
- [ ] **PDV + recibo**: venda em qualquer formato (inclusive por peso), baixa de
      estoque de produto acabado (respeitando lote), geração de recibo imprimível/PDF.
- [ ] **Relatórios**: estoque + alertas de estoque mínimo e de validade próxima;
      **custo x preço praticado x margem** por produto/formato.

Fica **fora** do MVP (fases futuras — ver doc 03):

- Nota fiscal eletrônica de **saída** (NFC-e/NF-e emitida na venda), SPED, contador.
- Integração com balança, impressora térmica, gaveta, TEF/maquininha.
- Recall/rastreabilidade completa de lote ponta a ponta.
- App mobile dedicado / modo offline.
- Curva ABC, previsão de demanda, sazonalidade, promoções.
- Financeiro completo (contas a pagar/receber, fluxo de caixa, DRE).
- Delivery / cardápio online / integração iFood.

## 6. Requisitos não-funcionais

- **Multi-tenant** desde o início (isolamento por empresa). Ver doc 02.
- **Web responsivo** (balcão em tablet/PC + escritório). Sem app nativo no MVP.
- **Português (pt-BR)**; valores em **R$**; datas em pt-BR.
- **Precisão:** dinheiro/quantidade em `Decimal` (nunca `float`).
- **Auditoria mínima:** quem criou/alterou lançamentos de estoque e vendas.
- **Segurança:** senhas com hash; toda query filtrada por `empresaId`.

## 7. Decisões já tomadas com o cliente

- **Plataforma:** Web app.
- **Escopo comercial:** multi-empresa.
- **Stack:** Next.js + TypeScript + Prisma + PostgreSQL + Tailwind CSS.

## 8. Decisões definidas nesta rodada (respostas do cliente)

1. **Formatos de venda:** o sistema suporta **todas as formas** —
   fração (bola, casquinha, cascão), **self-service por kg**, e itens fechados
   (pote, picolé, balde). Modelado via `FormatoVenda` (doc 02).
2. **Atacado:** sim, há **venda de balde no atacado**. Cada formato tem preço de
   **varejo e atacado**.
3. **Lote e validade:** **sim, necessário** — no MVP, para produtos fabricados e
   insumos perecíveis.
4. **Custo da calda base + produto acabado:** requisito central; resolvido com
   **ficha técnica multi-nível** (calda base é um INTERMEDIÁRIO com custo próprio,
   reaproveitado por todos os sabores). Ver doc 02, seção 5.
5. **Importação de NF:** **XML (NF-e) e foto (OCR)**, com mapeamento
   produto-do-fornecedor → insumo memorizado. Ver doc 02, seção 7, e doc 03.
6. **Preços praticados e margem:** registrar preço praticado na venda e exibir
   **margem = preço − custo** por produto/formato nos relatórios.

## 9. Decisões ainda em aberto (menores, resolver durante a execução)

1. **Método de custeio padrão:** última compra ou custo médio ponderado?
   (Recomendo média ponderada como padrão, exibindo os dois valores.)
2. **Recibo:** algum layout/obrigação específica, ou comprovante simples (itens,
   valores, data, dados da empresa) basta no MVP?
3. **Precificação:** preço digitado manualmente ou o sistema **sugere preço** a
   partir do custo + margem desejada? (Recomendo sugerir e permitir editar.)
4. **Baixa por lote na venda:** automática por **FEFO** (vence primeiro, sai
   primeiro) ou o operador escolhe o lote? (Recomendo FEFO automático.)
5. **OCR da foto:** qual serviço/modelo usar para extrair os itens da imagem da
   nota (ver opções no doc 03, seção de importação de NF).
