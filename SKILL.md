---
name: integridade-coleta
description: Auditoria de integridade da coleta de dados em contas de mídia paga (Google Ads, Meta Ads, GA4). Verifica se as conversões estão instaladas, vivas e medindo o valor certo, se os anúncios estão parametrizados e se os números batem entre plataforma e Analytics. Entrega veredito de "conta apta para trabalhar", score de coleta ponderado por gasto e plano de ação priorizado. Use ao receber uma conta nova, no onboarding de cliente, antes de confiar em qualquer análise de performance, ou quando os números de uma plataforma não batem com o Analytics.
---

# Integridade da Coleta

Sanity check de tracking numa conta de mídia paga. Responde uma pergunta e só uma:
**dá para confiar nos dados dessa conta o suficiente para trabalhar nela?**

## O que esta skill NÃO é

Manter esta fronteira é o que faz a skill ser útil. Ela não faz:

- **Auditoria de performance.** Nada de desperdício, search terms, fadiga de criativo, sobreposição
  de público, Quality Score. Isso é outro pilar e existem outras skills para ele.
- **Auditoria de estrutura.** Nada de opinião sobre nomenclatura, quantidade de campanhas, PMax vs
  Search, Advantage+ vs manual.
- **Auditoria de web analytics.** Não inspeciona GTM, não lê container, não abre o site, não verifica
  dataLayer. Se um problema exigir isso, a skill **aponta o sintoma e manda o analista investigar** —
  ela não investiga por ele.
- **Execução.** Não corrige nada, não altera nada. O output é recomendação. Aceite e execução são
  responsabilidade de quem lê.

Quando um achado precisar de algo fora deste escopo, diga isso explicitamente no output em vez de
adivinhar. "Discrepância compatível com duplo disparo — verifique o container no GTM" é uma resposta
completa. Inventar a causa raiz sem ter olhado o site não é.

## Passo 1 — Escolher o modo

**Sempre pergunte antes de rodar.** Não assuma.

> Você quer a auditoria **geral** (rápida, em cima dos exports que você já tem, sem pedir acesso a
> nada) ou a **profunda** (conectada nas APIs, cobre configuração de conversão e varre anúncio por
> anúncio)?

### Modo geral

Não pede credencial nenhuma. O analista aponta ou cola o que já tem em mãos:

- Export de anúncios do Meta **com a coluna de URL/parâmetros** e gasto por anúncio
- Export de campanhas do Google Ads com gasto, conversões e valor de conversão
- Export ou print do relatório de aquisição do GA4 (origem/mídia, sessões, receita)

Rode **apenas os checks que esses dados sustentam**. Todo check sem dado vira `NAO_VERIFICADO` —
nunca `PASSOU`. Um check não verificado não entra no cálculo do score e aparece na cobertura.

### Modo profundo

Conecta direto nas APIs — **não** via Windsor, Dataslayer ou Supermetrics. Esses conectores são
camada de relatório: entregam número, mas são cegos para configuração de conversão, que sustenta
metade deste checklist. Confirmado: o Dataslayer expõe `url_tags`/`link_url` do Meta (parametrização
dá para ver), mas no Google Ads só entrega `ConversionTrackerId` e `ConversionTypeName` — nome como
dimensão de relatório, nada de status, contagem, valor ou atribuição.

Inputs necessários:

| Fonte | Input | Ferramenta |
|---|---|---|
| Google Ads | `customer_id` | `run_gaql` (MCP google-ads) |
| Meta Ads | `act_<ad_account_id>` | MCP fb-ads |
| GA4 | `property_id` | GA4 Data API |

Se uma fonte faltar, **rode mesmo assim**. Ausência de fonte é pré-condição não atendida, não motivo
para abortar. Sem GA4, todos os checks de discrepância viram `NAO_VERIFICADO` e a cobertura cai —
diga isso no output, com todas as letras, porque uma auditoria sem árbitro é uma auditoria fraca e o
leitor precisa saber disso.

## Passo 2 — Detectar o perfil da conta

Antes de rodar check nenhum, estabeleça duas coisas, porque elas ligam e desligam pré-condições:

1. **A conta tem receita?** Existe conversão com valor monetário significativo (compra, receita,
   purchase)? Isso liga os checks de valor e de discrepância.
2. **Quais fontes estão disponíveis?** Google, Meta, GA4 — quais dessas três você consegue ler?

Registre o resultado. Ele determina quais checks são aplicáveis e é o que torna o score honesto.

## Passo 3 — Janelas de tempo

Duas janelas, dois propósitos. Não misture.

- **Janela de medição** — `últimos 30 dias, terminando 3 dias atrás`. Use para discrepância,
  parametrização, ponderação por gasto. **Os 3 dias de corte não são negociáveis**: as plataformas
  preenchem conversão retroativamente dentro da janela de atribuição, e comparar dado fresco contra
  o GA4 gera discrepância falsa em conta perfeitamente saudável. É a causa nº 1 de falso positivo
  nesse tipo de auditoria.
- **Janela de vitalidade** — `últimos 7 dias corridos`, sem corte. Use só para "chegou alguma
  coisa?". Aqui a pergunta não é se o número bate, é se existe número.

## Passo 4 — Rodar o checklist

Queries e chamadas em `references/queries.md`. Limiares em `references/limiares.md`.

Cada check tem: um **id**, uma **pré-condição**, um **tipo** (invariante ou diretriz) e um **escopo
de gasto** (quanto do investimento aquele check cobre).

- **Invariante** = não pode não ter. Violação é erro, independente de tamanho. Alimenta o veredito.
- **Diretriz** = deveria ser assim. Violação é recomendação, não reprova a conta.

Se a pré-condição não for atendida, o resultado é `NAO_APLICAVEL` (a conta não tem aquilo) ou
`NAO_VERIFICADO` (a conta pode ter, mas você não conseguiu ler). **Os dois são diferentes e nenhum
dos dois é `PASSOU`.** Não aprove o que você não olhou.

### Google Ads

| id | Check | Tipo | Pré-condição | Escopo de gasto |
|---|---|---|---|---|
| `G01` | Existe ao menos uma conversion action com `status = ENABLED` e `include_in_conversions_metric = TRUE` | Invariante | Google acessível | 100% do gasto Google |
| `G02` | Essa conversion action registrou conversão na janela de vitalidade | Invariante | `G01` passou | 100% do gasto Google |
| `G03` | Valor de conversão presente e plausível — não zero, não `always_use_default_value` com valor simbólico, ticket médio compatível com o negócio | Invariante | Conta tem receita | 100% do gasto Google |
| `G04` | `counting_type` coerente com a categoria: `MANY_PER_CLICK` para compra, `ONE_PER_CLICK` para lead | Diretriz | `G01` passou | 100% do gasto Google |
| `G05` | `customer.auto_tagging_enabled = TRUE` (GCLID ligado) | Invariante | Google acessível | 100% do gasto Google |
| `G06` | Nenhum `tracking_url_template` ou `final_url_suffix` sobrescrevendo ou neutralizando o GCLID | Invariante | `G05` passou | Gasto das campanhas afetadas |
| `G07` | Não há conversão importada do GA4 (`GOOGLE_ANALYTICS_4_*`) **e** tag nativa da mesma categoria, ambas com `include_in_conversions_metric = TRUE` | Invariante | `G01` passou | 100% do gasto Google |

Notas de julgamento:

- **`G03`** é o check do "compra de R$ 1". Compare o valor médio por conversão contra o ticket médio
  real do negócio (pergunte se não souber). Valor unitário fixo em toda conversão é o sinal clássico
  de tag disparando sem `value` dinâmico.
- **`G06`** é o caso concreto que motivou esta skill: parametrização manual do Google Ads convivendo
  com auto-tagging. Muitas vezes não chega a quebrar nada — mas é inútil e só gera risco. Reporte
  mesmo quando o dado ainda parecer sadio.
- **`G07`** conta a mesma compra duas vezes e infla ROAS silenciosamente. Comum e caro.

### Meta Ads

| id | Check | Tipo | Pré-condição | Escopo de gasto |
|---|---|---|---|---|
| `M01` | Pixel com evento de conversão principal registrando na janela de vitalidade | Invariante | Meta acessível | 100% do gasto Meta |
| `M02` | CAPI presente e enviando eventos | Diretriz | Meta acessível | 100% do gasto Meta |
| `M03` | Todo anúncio ativo tem `url_tags` preenchido | Invariante | Meta acessível | Gasto dos anúncios sem parâmetro |
| `M04` | Os parâmetros são consistentes entre anúncios — mesmo padrão de `utm_source`/`utm_medium`, sem variação livre | Invariante | `M03` verificado | Gasto dos anúncios divergentes |

Notas de julgamento:

- **`M02` é diretriz, não invariante.** Não ter CAPI é uma perda, não uma mentira: a conta ainda
  está medindo, só está medindo menos. Recomende, não reprove.
- **Dedup de CAPI não é um check direto e isso é deliberado.** A Marketing API não expõe Event Match
  Quality nem taxa de deduplicação de forma confiável — eles vivem no Events Manager. Falha de dedup
  se manifesta como receita do Meta desproporcional ao GA4, e é `X02` que pega isso. Quando `X02`
  estourar, cite dedup como hipótese e mande verificar no Events Manager. **Nunca afirme que a dedup
  está quebrada — você não tem como saber.**
- **`M04`** é a bagunça do "Meta Paid, Instagram Paid, Facebook Paid, Facebook CPC". Cada variante
  vira uma linha separada no GA4 e a conta perde a capacidade de somar a própria mídia.

### GA4

| id | Check | Tipo | Pré-condição | Escopo de gasto |
|---|---|---|---|---|
| `A01` | GA4 recebe `google / cpc` com volume compatível com o gasto do Google | Invariante | GA4 + Google acessíveis | 100% do gasto Google |
| `A02` | Tráfego do Meta chega consolidado — não fragmentado em múltiplas variantes de origem/mídia | Invariante | GA4 + Meta acessíveis | 100% do gasto Meta |

### Cross-source

| id | Check | Tipo | Pré-condição | Escopo de gasto |
|---|---|---|---|---|
| `X01` | Receita reportada pelo Google dentro da faixa aceitável contra `google / cpc` no GA4 | Invariante | GA4 + Google acessíveis, conta tem receita | 100% do gasto Google |
| `X02` | Receita reportada pelo Meta dentro da faixa aceitável contra o tráfego pago do Meta no GA4 | Invariante | GA4 + Meta acessíveis, conta tem receita | 100% do gasto Meta |

**Não compare Google contra Meta diretamente.** Eles atribuem por janelas e critérios diferentes —
view-through no Meta, last-click no Google. Divergir é o comportamento esperado, não o defeito. O
GA4 é o árbitro; sem ele, não há comparação válida, e `X01`/`X02` viram `NAO_VERIFICADO`.

## Passo 5 — Score e veredito

São **dois eixos independentes**. Um número só não responde às duas perguntas.

### Score de coleta

Não conta itens. Conta dinheiro.

```
gasto_analisado    = gasto total das fontes que você conseguiu ler, na janela de medição
gasto_comprometido = gasto coberto por ao menos UMA invariante violada (união, sem dupla contagem)
score              = 100 × (1 − gasto_comprometido / gasto_analisado)
```

Um score de 95% significa **"95% do investimento está sendo medido corretamente"** — uma frase com
significado operacional. Não significa "17 de 18 checks passaram", que não significa nada.

A união é o que resolve o caso do anúncio de 1%: um anúncio sem parâmetro representando 1% do gasto
tira 1 ponto do score e é listado como violação. Ele está errado — e um anúncio errado não pode pesar
como uma tag de compra morta.

Sempre publique a **cobertura** ao lado do score: quantos checks aplicáveis foram efetivamente
verificados. Score de 100% com cobertura de 40% não é uma conta saudável, é uma auditoria cega, e o
output tem que deixar isso impossível de confundir.

### Veredito

| Veredito | Condição |
|---|---|
| `NAO_APTA` | Gasto comprometido por invariantes ≥ **10%** do gasto analisado |
| `APTA_COM_RESSALVAS` | Alguma invariante violada, abaixo do limiar de materialidade |
| `APTA` | Nenhuma invariante violada (diretrizes podem estar violadas) |

Diretriz violada **nunca** move o veredito.

O limiar de 10% se autorregula: uma tag de compra morta compromete 100% do gasto daquela plataforma
e reprova sozinha, como deve. Um anúncio bobo não.

## Passo 6 — Output

Bloco estruturado primeiro, prosa depois. O bloco existe porque o destino deste output é um painel
com aceite de recomendação, e aceite precisa de um objeto com id — não de um parágrafo em português
para alguém extrair com regex depois.

````markdown
```json
{
  "conta": "<nome do cliente>",
  "modo": "geral | profundo",
  "janela_medicao": "2026-06-13 a 2026-07-13",
  "perfil": { "tem_receita": true, "fontes": ["google_ads", "meta_ads", "ga4"] },
  "veredito": "NAO_APTA",
  "score_coleta": 62,
  "cobertura": { "aplicaveis": 11, "verificados": 11 },
  "gasto_analisado": 48200.00,
  "gasto_comprometido": 18300.00,
  "achados": [
    {
      "id": "X02",
      "tipo": "invariante",
      "status": "VIOLADO",
      "severidade": "alta",
      "gasto_exposto": 18300.00,
      "evidencia": "Meta reporta R$ 250.000 de receita na janela; GA4 atribui R$ 170.000 ao tráfego pago do Meta. Delta de +47%.",
      "acao": "Verificar deduplicação de CAPI no Events Manager: eventos de navegador e servidor devem compartilhar event_id.",
      "fora_de_escopo": true
    }
  ]
}
```

## Integridade da Coleta — <Cliente>

**Veredito: NÃO APTA** · Score de coleta: **62%** · Cobertura: 11/11 checks

### Faça isso

1. **Verificar deduplicação de CAPI no Meta** — R$ 18.300 de gasto exposto
   <details><summary>Por quê</summary>

   O Meta reporta R$ 250.000 de receita na janela de medição. O GA4 atribui R$ 170.000 ao tráfego
   pago do Meta no mesmo período — um delta de +47%, acima da faixa normal de 60%... [etc]
   </details>

### Está tudo bem aqui

- `G05` Auto-tagging ligado, GCLID chegando íntegro
- ...

### Não deu para verificar

- ...
```
````

Regras do output:

- **A recomendação vem primeiro. O porquê fica recolhido.** É o Waze: a explicação existe para quem
  a quiser, não como pedágio para quem só quer saber o que fazer. Nunca abra com a análise e termine
  na conclusão.
- **Ordene por gasto exposto, decrescente.** É dinheiro, não é gosto.
- **Toda evidência carrega o número que a sustenta.** "Discrepância alta" não é evidência.
  "R$ 250.000 contra R$ 170.000, +47%" é.
- **Marque `fora_de_escopo: true`** quando a ação exigir GTM, site ou desenvolvedor. O analista
  precisa saber, antes de abrir o item, que aquilo não se resolve dentro da plataforma.
- **Diga o que está bem.** Uma auditoria que só lista defeito não é lida duas vezes.

## Guias

- **Nunca marque `PASSOU` no que você não verificou.** Este é o único jeito de a skill perder a
  confiança do analista de forma irrecuperável. `NAO_VERIFICADO` é uma resposta honesta e útil;
  aprovação falsa não.
- **Sintoma não é diagnóstico.** Você mede discrepância. Você não viu a tag disparar. Escreva
  "compatível com", "verifique", "provável" — não "a tag está disparando duas vezes".
- **Pergunte o ticket médio** se `G03` for aplicável e você não tiver referência. Chutar plausibilidade
  de valor sem saber o que a loja vende é como gerar falso positivo de propósito.
- **Se o modo geral não trouxer gasto por anúncio**, `M03` não pode ser ponderado — reporte a
  contagem de anúncios sem parâmetro e marque o check como verificado sem peso de gasto, declarando
  a limitação. Não invente rateio.
- **Os limiares vão mudar com o uso.** Eles estão em `references/limiares.md` justamente para serem
  calibrados sem tocar nesta lógica. Se um limiar gerar falso positivo recorrente, o problema é o
  limiar.
