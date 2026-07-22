# 04 — Análise de Concorrência

> Pesquisa de mercado feita para orientar posicionamento e escopo. Resume os
> concorrentes avaliados, o que aprender com cada um e onde está a nossa brecha.
> Fontes ao final.

## 1. Mapa do mercado

O mercado se divide em dois campos, e há uma **brecha no meio** — que é onde
miramos:

- **Camp A — PDV-verticais de sorveteria:** fortes no balcão (venda por peso/bola,
  delivery, fiscal, cardápio), mas **rasos em fabricação**. Tratam "ficha técnica"
  apenas como *baixa de insumo na venda*, não como manufatura com semi-acabado
  (calda base) e custo real de produção.
  Ex.: **SisFood, Saipos, Food Sistemas, Nex, Nox**.
- **Camp B — ERPs industriais:** fortes em produção, receitas, lote/validade, CMV e
  franquia — mas **pesados, complexos e caros**, voltados a média/grande indústria.
  Ex.: **RTSys SGI, WebMais**.
- **Horizontais com produção:** ERPs genéricos com módulo de produção + import XML
  (**GestãoClick, Tiny, Omie**) e ERP/PDV de varejo (**GranMoney**).

**Nossa brecha:** a **sorveteria-fábrica pequena/média que produz E vende** e
precisa de **custo de calda base + ficha multi-nível + import de NF fácil + PDV com
formatos**, num pacote **simples e acessível**. Camp A é raso na fábrica; Camp B é
overkill. Ninguém une profundidade de fabricação com simplicidade e preço.

## 2. Concorrentes avaliados

### GranMoney — ERP/PDV de varejo horizontal
- **Forte:** PDV **offline com sync**, emissão fiscal completa (NF-e/NFC-e/NFS-e/
  CF-e/MF-e), financeiro (contas a pagar/receber), comanda eletrônica, integração
  iFood, preço modular (a partir de ~R$50/mês + add-ons), marca madura.
- **Fraco p/ nós:** sem ficha técnica multi-nível, sem produção/fabricação, sem
  custo de calda base. É venda + fiscal, não fábrica.

### 1. RTSys SGI — ERP para fábricas de sorvete *(mais próximo do nosso lado fábrica)*
- **Forte:** produção industrial, receitas, **lote/validade, custo do produto
  acabado**, gestão de **franquias** e da cadeia inteira (indústria → CD → loja →
  franquia → cliente); pedido da loja vira ordem de separação/expedição/faturamento.
- **Crítica:** para **rede/indústria de médio-grande porte**. Complexo, caro,
  curva de aprendizado alta. Canhão para uma sorveteria de uma unidade.

### 2. WebMais Sistemas — ERP indústria de sorvetes/picolés
- **Forte:** **ordem de produção a partir dos pedidos de venda**, ficha técnica
  única, lote/validade, **CMV e formação de preço com margem**, relatórios de
  produção (funcionário, tempo, consumo de MP), 100% nuvem.
- **Crítica:** ERP industrial genérico "adaptado" a sorvete; peso e complexidade de
  indústria. Foco em quem já é fábrica estruturada.

### 3. SisFood — vertical sorveteria/gelateria/açaiteria *(mais direto no balcão)*
- **Forte:** **ficha técnica com baixa automática de insumo na venda** (bola,
  leite, calda, complementos), **CMV em tempo real**, **venda por peso/self-service**,
  alerta de validade e de **estoque mínimo/ponto de pedido**, cardápio digital,
  delivery, **já adaptado à Reforma Tributária 2026**.
- **Crítica:** a ficha técnica é para **baixar insumo na venda**, não para
  **fabricar**. Não modela a **calda base como semi-acabado com custo próprio**
  produzida em lote — ou seja, não entrega o **custo real de produção** (nossa dor #2).

### 4. Saipos — PDV food service robusto
- **Forte:** **balança integrada** (peso→preço), **selo iFood Super Integrador**,
  delivery forte, marca consolidada.
- **Crítica:** **reputação com ressalvas** (Reclame Aqui ~7,4; ~390 reclamações):
  **quedas do sistema**, **cancelamento difícil**, cobranças. É PDV/delivery, não é
  fabricação. Lição dupla: produto **e** confiança.

### 5. Food Sistemas — vertical sorveteria com marketing de IA/IoT
- **Forte (na proposta):** baixa por **grama/ml** (sundae baixa sorvete + calda +
  copo + colher), **alertas inteligentes** ("no ritmo atual o pistache acaba
  sábado"), **sugestão automática de compra**, **curva ABC**, lote/validade,
  cadeia fria.
- **Crítica:** muito **buzzword** ("motor climático com IA", "IoT anti-derretimento").
  A base é boa, mas parte é marketing. O recurso mais copiável é a **inteligência de
  estoque** (previsão de ruptura + sugestão de compra), sem o exagero.

## 3. O que vamos adotar (mapeado ao plano)

| # | Ideia (de quem) | O que fazer | Onde |
|---|---|---|---|
| 1 | **Baixa de insumo na venda de itens montados** (SisFood, Food) | **Composição de item de venda**: sundae/milkshake/açaí baixam vários insumos, reusando `Receita`/`ComponenteReceita` na venda. | doc 02 §4A/§6.4, Fase 6 |
| 2 | **CMV em tempo real** (SisFood, WebMais) | CMV como métrica de primeira classe. | Fase 7 |
| 3 | **Previsão de ruptura + sugestão de compra** (Food, SisFood) | Consumo médio → "acaba em X dias" + ponto de pedido. Simples, sem hype. | Fase 7 |
| 4 | **Balança integrada** (Saipos, SisFood) | Self-service por kg já previsto; adicionar ponto de integração com balança (peso digitado → leitura). | Fase 6 + backlog |
| 5 | **Ordem de produção sugerida pela demanda** (WebMais, RTSys) | "Produza calda base — vai acabar". | Backlog |
| 6 | **Curva ABC de insumos** (Food) | Relatório barato de alto valor. | Backlog |
| 7 | **Pedido loja→fábrica (B2B interno)** (RTSys) | Multi-tenant + atacado: pontos de venda pedem à fábrica. | Backlog |
| 8 | **Cardápio digital / delivery / iFood** (SisFood, Saipos) | Table stakes de fast-follow. | Fase 8 |
| 9 | **Confiabilidade + billing transparente** (fraqueza da Saipos) | PDV offline-first + cobrança/cancelamento honestos = diferencial de confiança. | Estratégia + Fase 8 |
| 10 | **Reforma Tributária 2026** (SisFood) | Ao entrar no fiscal, já nascer nas novas regras. | Fase 8 |

## 4. Posicionamento

- **Não competir em amplitude** (fiscal, financeiro, delivery) com quem tem anos de
  vantagem. **Ganhar em profundidade vertical de fabricação**: calda base com custo
  próprio + ficha multi-nível + custo em cascata + import de NF com mapeamento +
  lote/validade + formatos de venda.
- **MVP forte em produção/custo** (nossa vantagem única) e depois **fast-follow nos
  table stakes** (PDV offline → fiscal → delivery).
- **Confiabilidade e transparência** como marca (contraponto às dores de Saipos).

## 5. Fontes

- GranMoney — https://granmoney.com/ ; https://granmoney.com/frente-de-caixa-gratuito ; https://granmoney.com/ifood-pdv ; https://granmoney.com/calculate-form
- RTSys SGI — https://www.rtsys.com.br/erp-sistema-gestao-fabricas-de-sorvetes-sgi-rtsys/
- WebMais — https://webmaissistemas.com.br/erp-para-industria-de-sorvetes/
- SisFood — https://www.sisfood.com.br/segmentos/sistema-para-sorveteria
- Saipos — https://saipos.com/sistema/sorveteria ; Reclame Aqui: https://www.reclameaqui.com.br/empresa/saipos/
- Food Sistemas — https://foodsistemas.com.br/segmentos/sistema-para-sorveteria/
- GestãoClick (import XML) — https://ajuda.gestaoclick.com.br/hc/pt-br/articles/360047287814-Importe-XML-de-nota-fiscal-de-compra

> Pesquisa realizada em jul/2026. Preços e recursos de terceiros mudam; revalidar
> antes de decisões comerciais.
