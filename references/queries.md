# Queries e chamadas

Mecânica de leitura da skill `integridade-coleta`, modo profundo. Cada bloco mapeia para os ids de
check do `SKILL.md`.

**Regra geral:** os nomes de campo abaixo são os da API na data de escrita. Se uma query falhar por
campo inexistente, **não invente substituto** — reporte o check como `NAO_VERIFICADO` e diga qual
campo faltou. Um check honestamente não verificado vale mais que um chute.

---

## Google Ads — MCP `google-ads-mcp-server`, tool `run_gaql`

### `G01`, `G03`, `G04`, `G07` — configuração das conversion actions

```sql
SELECT
  conversion_action.id,
  conversion_action.name,
  conversion_action.status,
  conversion_action.type,
  conversion_action.category,
  conversion_action.counting_type,
  conversion_action.include_in_conversions_metric,
  conversion_action.primary_for_goal,
  conversion_action.value_settings.default_value,
  conversion_action.value_settings.always_use_default_value,
  conversion_action.click_through_lookback_window_days,
  conversion_action.attribution_model_settings.attribution_model
FROM conversion_action
```

Como ler:

- **`G01`** — precisa existir ao menos uma linha com `status = ENABLED` **e**
  `include_in_conversions_metric = TRUE`. Uma conversão que existe mas não entra na coluna
  "Conversões" não está guiando o algoritmo: para efeito de lance, ela não existe.
- **`G03`** — `always_use_default_value = TRUE` numa conversão de receita é violação direta: toda
  compra está sendo lançada pelo mesmo valor. Confirme com o valor médio real (query de vitalidade
  abaixo).
- **`G04`** — `counting_type`: `MANY_PER_CLICK` é o correto para compra (o mesmo clique pode comprar
  duas vezes), `ONE_PER_CLICK` para lead. Compra em `ONE_PER_CLICK` subconta.
- **`G07`** — o check da dupla contagem. Procure `type` começando com `GOOGLE_ANALYTICS_4_`
  convivendo com uma conversão nativa (`WEBPAGE`) de **mesma `category`** (ex.: `PURCHASE`), ambas
  com `include_in_conversions_metric = TRUE`. Isso conta a mesma compra duas vezes e infla ROAS
  silenciosamente.

### `G02`, `G03` — vitalidade e valor real por conversão

```sql
SELECT
  segments.conversion_action_name,
  segments.conversion_action,
  metrics.all_conversions,
  metrics.all_conversions_value
FROM customer
WHERE segments.date DURING LAST_7_DAYS
```

- **`G02`** — `all_conversions` deve ser > 0 para a conversão principal identificada em `G01`.
- **`G03`** — `all_conversions_value / all_conversions` = valor médio real. Compare contra o ticket
  médio informado pelo analista (limiares em `limiares.md`). Sem ticket médio, `NAO_VERIFICADO`.

Para a janela de medição, troque o `WHERE` por datas explícitas:

```sql
WHERE segments.date BETWEEN '2026-06-13' AND '2026-07-13'
```

### `G05` — auto-tagging e setting de conversão

```sql
SELECT
  customer.id,
  customer.descriptive_name,
  customer.auto_tagging_enabled,
  customer.conversion_tracking_setting.conversion_tracking_id,
  customer.conversion_tracking_setting.google_ads_conversion_customer
FROM customer
```

`auto_tagging_enabled = FALSE` é invariante violada e compromete 100% do gasto do Google: sem GCLID,
nada do que vem depois é confiável.

`google_ads_conversion_customer` diferente do `customer.id` indica conversão gerenciada por outra
conta (MCC ou cross-account). Não é erro, mas muda onde investigar — mencione no output.

### `G06` — parametrização sobrepondo o GCLID

Nível de conta:

```sql
SELECT
  customer.id,
  customer.tracking_url_template,
  customer.final_url_suffix
FROM customer
```

Nível de campanha:

```sql
SELECT
  campaign.id,
  campaign.name,
  campaign.status,
  campaign.tracking_url_template,
  campaign.final_url_suffix,
  campaign.url_custom_parameters,
  metrics.cost_micros
FROM campaign
WHERE campaign.status = 'ENABLED'
  AND segments.date BETWEEN '2026-06-13' AND '2026-07-13'
```

Como ler:

- `tracking_url_template` contendo redirect de terceiro sem repassar `{gclid}` ou `{lpurl}` → GCLID
  se perde. Violação.
- `final_url_suffix` com `utm_*` manual numa conta com auto-tagging ligado → normalmente **não
  quebra**, mas é redundante e cria risco de conflito. Reporte mesmo assim: foi exatamente o caso que
  motivou esta skill.
- `cost_micros / 1e6` = gasto da campanha, para ponderar o escopo do check.

---

## Meta Ads — MCP `fb-ads-mcp-server`

### `M03`, `M04` — parametrização anúncio a anúncio

`get_ads_by_adaccount` no `act_<id>`, pedindo os campos de criativo. O que interessa:

| Campo | Uso |
|---|---|
| `creative.url_tags` | Os parâmetros de fato. Vazio = `M03` violado. |
| `creative.object_story_spec.link_data.link` | URL de destino; pode conter parâmetros inline. |
| `creative.template_url_spec` | Sobrescrita de URL em alguns formatos. |
| `effective_status` | Filtre por `ACTIVE` — anúncio pausado não gasta, não pesa. |

Gasto por anúncio vem de `get_ad_insights` na janela de medição, para ponderar `M03`/`M04`.

Como ler:

- **`M03`** — `url_tags` vazio num anúncio ativo com gasto = violação, ponderada pelo gasto **daquele
  anúncio**. Cuidado: parâmetros podem estar inline no `link` em vez de em `url_tags`. Cheque os dois
  antes de acusar.
- **`M04`** — normalize os `utm_source`/`utm_medium` encontrados e conte as variantes distintas. Mais
  de um par por plataforma é a bagunça do "Meta Paid / Instagram Paid / Facebook CPC". Pondere pelo
  gasto dos anúncios divergentes do padrão dominante.

### `M01`, `M02` — pixel e CAPI

`get_ad_insights` na janela de vitalidade, com as ações de conversão:

- **`M01`** — a ação de conversão principal (`purchase` / `offsite_conversion.fb_pixel_purchase`)
  deve ter contagem > 0.
- **`M02`** — presença de CAPI é inferível pela existência de eventos de servidor. **Não force.** Se
  a API não der um sinal claro, marque `NAO_VERIFICADO` e peça ao analista para conferir no Events
  Manager. `M02` é diretriz — não ter certeza aqui não compromete o veredito.

**Dedup:** não existe query aqui, e isso é por design. Event Match Quality e taxa de dedup vivem no
Events Manager e não são expostos de forma confiável pela Marketing API. O sintoma é pego por `X02`.

### Alternativa via Dataslayer

Se o MCP do fb-ads não estiver disponível, o Dataslayer entrega a parte de parametrização —
confirmado: datasource `facebook_ads`, dimensões `url_tags`, `link_url`, `object_story_spec_link`,
`template_url`. Serve para `M03`/`M04`.

**Não serve para Google.** O `google_ads` do Dataslayer só expõe `ConversionTrackerId` e
`ConversionTypeName` — o nome da conversão como dimensão de relatório. Nada de `status`,
`counting_type`, `value_settings` ou `include_in_conversions_metric`. Ou seja: `G01`, `G03`, `G04`,
`G05`, `G06` e `G07` são **impossíveis** por conector. É a razão pela qual o modo profundo vai direto
na API.

---

## GA4 — Data API

### `A01`, `A02`, `X01`, `X02` — origem/mídia e receita

Relatório na janela de medição:

| | |
|---|---|
| Dimensão | `sessionSourceMedium` |
| Métricas | `sessions`, `purchaseRevenue`, `transactions` |

Como ler:

- **`A01`** — `google / cpc` precisa existir com volume de sessões compatível com os cliques do
  Google no mesmo período. Ausência total com gasto ativo = invariante violada, e provavelmente o
  auto-tagging está desligado (cruze com `G05`).
- **`A02`** — agrupe tudo que for Meta (`facebook`, `fb`, `instagram`, `ig`, `meta` em qualquer
  combinação de source/medium). Mais de uma variante relevante = violação.
- **`X01`** — `purchaseRevenue` de `google / cpc` é o denominador do delta contra a receita do Google
  Ads.
- **`X02`** — `purchaseRevenue` somado de todas as variantes do Meta é o denominador do delta contra
  a receita do Meta. **Some as variantes antes de comparar** — comparar contra uma variante só
  produz discrepância inventada, que é falso positivo criado pela própria skill.

---

## Modo geral — exports

Sem API. O que cada export sustenta:

| Export | Checks que habilita |
|---|---|
| Meta, nível de anúncio, com coluna de URL/parâmetros + gasto | `M03`, `M04` |
| Meta, nível de conta, com conversões e receita | `M01`, `X02` (parcial) |
| Google Ads, nível de campanha, com custo, conversões e valor | `G03` (parcial), `X01` (parcial) |
| GA4, aquisição por origem/mídia, com sessões e receita | `A01`, `A02`, `X01`, `X02` |

**Nenhum export sustenta `G01`, `G02`, `G04`, `G05`, `G06` ou `G07`.** Toda a configuração de
conversão do Google é invisível fora da API — export de relatório não carrega configuração de conta.
No modo geral, esses seis são sempre `NAO_VERIFICADO`, a cobertura cai para perto da metade, e o
output tem que dizer isso na primeira linha. É a diferença honesta entre os dois modos, e escondê-la
transformaria o modo geral numa máquina de aprovar conta quebrada.
