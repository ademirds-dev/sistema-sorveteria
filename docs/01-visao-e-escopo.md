# 01 — Visão, Público e Escopo

> Documento de planejamento. Nenhuma linha deste arquivo é código de execução —
> é a fonte de verdade sobre **o que** o sistema faz e **para quem**.

## 1. Visão

Sistema de gestão para **sorveteria que também é fábrica de sorvete** (produz o
próprio sorvete e vende no varejo e/ou atacado). O foco não é só o ponto de venda:
é conectar **compra de insumo → custo real → ficha técnica → produção → estoque →
venda → recibo → margem de lucro**.

O sistema nasce para resolver dores concretas de uma sorveteria com ~40 anos de
operação, mas é construído desde o início como **SaaS multi-empresa (multi-tenant)**,
para no futuro ser vendido a outras sorveterias.

## 2. Dores que originaram o projeto (declaradas pelo cliente)

1. **Controle de estoque** de insumos.
2. **Custo do produto sempre atualizado**, recalculado a partir das últimas
   compras de insumo.
3. **Emissão de recibo** de venda.
4. **Divisão de insumos por fabricação** — quanto de cada insumo entra em cada
   sabor/tipo de sorvete (ficha técnica / receita).

Essas quatro dores são o **coração do MVP**. Todo o resto é priorizado em volta delas.

## 3. Personas (quem usa)

| Persona | Papel no sistema | Precisa de |
|---|---|---|
| **Dono / Gerente** | administra tudo | cadastros, custos, relatórios, margem |
| **Produção / Fábrica** | fabrica os lotes | fichas técnicas, registrar produção, ver estoque de insumo |
| **Caixa / Balconista** | vende no balcão | PDV simples, gerar recibo |

No sistema isso vira **papéis de acesso (roles)**: `OWNER`, `GERENTE`, `PRODUCAO`,
`CAIXA`. Cada empresa (tenant) tem seus próprios usuários.

## 4. Conceitos do domínio (glossário)

Estes termos aparecem no modelo de dados e nas telas. Alinhá-los evita retrabalho.

- **Insumo (matéria-prima):** tudo que é comprado para produzir ou vender. Inclui
  ingredientes (leite, creme, açúcar, glucose, estabilizante, emulsificante, polpa
  de fruta, chocolate, castanha, essência) **e embalagens** (pote, tampa, colher,
  casquinha, saco, balde). Embalagem também tem custo e entra no custo do produto.
- **Unidade de medida + conversão:** compra-se em uma unidade (saco de 25 kg,
  caixa, litro) e consome-se em outra (gramas, ml) na receita. **Conversão de
  unidade é obrigatória** — é uma dor real (comprar açúcar em saco de 50 kg e usar
  em gramas por receita).
- **Fornecedor:** de quem se compra o insumo.
- **Compra de insumo:** entrada de estoque. Registra quantidade, valor pago,
  fornecedor e data. É o que **atualiza o custo** do insumo.
- **Método de custeio:** como o custo do insumo é calculado a partir das compras.
  Ver decisão no doc 02. Suportar **custo da última compra** (pedido do cliente) e
  **custo médio ponderado móvel** (mais justo), configurável por empresa.
- **Produto:** o que é vendido/fabricado. Tem **tipo** (ex.: sorvete de massa,
  picolé, açaí, soft, casquinha) e **sabor** (chocolate, morango, creme...).
- **Ficha técnica (receita):** lista de insumos e quantidades para produzir um
  lote de um produto. Base do cálculo de custo e da baixa de estoque na produção.
- **Rendimento:** quanto um lote produz (ex.: receita rende 10 L / 5 baldes).
- **Produção (lote):** ato de fabricar. Consome insumos (baixa estoque conforme
  ficha técnica) e gera **produto acabado** em estoque, com custo do lote e,
  opcionalmente, número de lote e validade.
- **Produto acabado:** estoque do que já foi fabricado e está pronto para vender.
- **Venda:** saída de produto acabado. Gera **recibo** e baixa o estoque.
- **Recibo:** comprovante simples da venda (não é nota fiscal). Nota fiscal
  (NFC-e/NF-e) é assunto de fase futura.

## 5. Escopo do MVP (versão 1)

Objetivo do MVP: **resolver as 4 dores declaradas, ponta a ponta, para uma empresa.**

Entra no MVP:

- [ ] Cadastro de **empresa (tenant)** + **usuários** com papéis + login.
- [ ] Cadastro de **fornecedores**.
- [ ] Cadastro de **insumos** com unidade de compra, unidade de uso e fator de
      conversão, estoque atual e estoque mínimo.
- [ ] **Compra de insumo** que dá entrada no estoque e **atualiza o custo**
      (última compra + custo médio ponderado).
- [ ] Cadastro de **produtos** (tipo + sabor) e **ficha técnica** associando
      insumos e quantidades por lote, com rendimento.
- [ ] **Cálculo automático do custo do produto** a partir da ficha técnica e do
      custo atual dos insumos.
- [ ] **Registrar produção** de um lote: baixa insumos do estoque, gera produto
      acabado e registra o custo do lote.
- [ ] **Venda** de produto acabado + **geração de recibo** (PDF/impressão) +
      baixa de estoque de produto acabado.
- [ ] **Painel/relatórios básicos:** estoque atual, alertas de estoque mínimo,
      custo e margem por produto.

Fica **fora** do MVP (fases futuras — ver doc 03):

- Nota fiscal eletrônica (NFC-e / NF-e), SPED, integração com contador.
- Integração com balança, impressora térmica, gaveta de dinheiro, TEF/maquininha.
- Controle de lote/validade rastreável ponta a ponta e recall.
- App mobile dedicado / modo offline.
- Curva ABC, previsão de demanda, sazonalidade, promoções.
- Financeiro completo (contas a pagar/receber, fluxo de caixa, DRE).
- Delivery / cardápio online / integração iFood.

## 6. Requisitos não-funcionais

- **Multi-tenant** desde o início (isolamento de dados por empresa). Ver doc 02.
- **Web responsivo** (balcão em tablet/PC + escritório). Sem app nativo no MVP.
- **Português (pt-BR)** em toda a interface; valores em **R$**; datas em pt-BR.
- **Precisão monetária:** nunca usar `float` para dinheiro/quantidade — usar
  `Decimal` (Prisma `Decimal` / Postgres `numeric`). Ver doc 02.
- **Auditoria mínima:** registrar quem criou/alterou lançamentos de estoque e vendas.
- **Segurança:** senhas com hash (bcrypt/argon2), toda query obrigatoriamente
  filtrada por `empresaId`.

## 7. Decisões já tomadas com o cliente

- **Plataforma:** Web app.
- **Escopo comercial:** multi-empresa (vai ser vendido a outras sorveterias).
- **Stack:** Next.js + TypeScript + Prisma + PostgreSQL + Tailwind CSS.

## 8. Decisões em aberto (precisam de resposta antes/durante a execução)

Estas perguntas mudam detalhes do modelo e das telas. Registrar a resposta aqui
quando decidido.

1. **Método de custeio padrão:** última compra ou custo médio ponderado? (Recomendo
   suportar os dois e deixar o médio ponderado como padrão, exibindo os dois valores.)
2. **Venda:** o balcão vende por **bola/casquinha** (fração de um pote) ou só por
   **item fechado** (pote, picolé, balde)? Isso muda a modelagem da venda a varejo.
3. **Atacado:** a fábrica vende **baldes para revenda** a outras lojas? Se sim, há
   tabela de preço diferente (varejo x atacado)?
4. **Recibo:** precisa de algum layout/obrigação específica, ou um comprovante
   simples com itens, valores, data e dados da empresa é suficiente no MVP?
5. **Lote/validade:** precisa rastrear lote e validade do produto acabado já no
   MVP, ou só o custo por enquanto?
6. **Precificação:** o preço de venda é digitado manualmente ou o sistema deve
   **sugerir preço** a partir do custo + margem desejada?
