# Limiares

Calibragem da skill `integridade-coleta`. **Este arquivo existe para ser editado.** Nenhum número
aqui é lei — todos são régua de quem roda conta, e régua se ajusta com o uso. Mexer aqui não deve
exigir tocar na lógica do `SKILL.md`.

## Discrepância entre plataforma e GA4

A régua é **diferente por plataforma**, e isso não é detalhe: Google e Meta atribuem por critérios
diferentes, então divergir do GA4 em graus diferentes é o comportamento correto de uma conta sadia.
Aplicar a mesma faixa nos dois é a forma mais rápida de encher o relatório de vermelho inútil.

Fórmula: `delta = (receita_plataforma − receita_ga4) / receita_ga4`

### `X01` — Google Ads vs `google / cpc` no GA4

| Faixa | Delta | Interpretação |
|---|---|---|
| Normal | até **+25%** | Janela de atribuição e modelagem de conversão explicam. Não reporte. |
| Alerta | **+25% a +60%** | Suspeito. Reporte como diretriz, não move o veredito. |
| Violação | acima de **+60%** | Invariante violada. Algo está contando errado. |

Régua mais apertada que a do Meta porque o Google, no default, atribui por clique e o GA4 vê o mesmo
clique. Divergência grande aqui tem menos desculpa legítima.

### `X02` — Meta Ads vs tráfego pago do Meta no GA4

| Faixa | Delta | Interpretação |
|---|---|---|
| Normal | até **+60%** | View-through de 1 dia infla legitimamente. Não reporte. |
| Alerta | **+60% a +120%** | Suspeito. Reporte como diretriz, não move o veredito. |
| Violação | acima de **+120%** | Invariante violada. |

Régua mais frouxa de propósito. O Meta credita conversão de quem viu e não clicou; o GA4 não tem como
enxergar essa pessoa. Uma conta perfeitamente instrumentada diverge aqui.

**Caso de referência:** conta reportando R$ 250.000 no Meta contra R$ 170.000 de tráfego pago do Meta
no GA4 = **+47%**, que cai em **alerta**, não em violação. É a calibragem certa: suspeito o bastante
para o analista ir investigar, não óbvio o bastante para reprovar a conta sozinho.

### Delta negativo

Plataforma reportando **menos** que o GA4 é anomalia de outra natureza — normalmente a tag está
subcontando ou a conta perdeu evento. Trate qualquer delta abaixo de **−20%** como alerta em ambas as
plataformas, independentemente da régua acima.

## Materialidade do veredito

| Parâmetro | Valor | Efeito |
|---|---|---|
| Limiar de reprovação | **10%** do gasto analisado comprometido por invariantes | Acima disso: `NAO_APTA` |

Se autorregula bem: uma tag de compra morta compromete 100% do gasto daquela plataforma e reprova
sozinha; um anúncio sem parâmetro representando 1% do gasto não.

## Vitalidade

| Parâmetro | Valor |
|---|---|
| Janela de vitalidade | **7 dias corridos**, sem corte |
| Mínimo para considerar viva | **≥ 1 conversão** registrada na janela |

Contas de volume muito baixo (menos de ~10 conversões/mês) dão falso positivo aqui. Se a conta tiver
volume baixo, estenda a janela de vitalidade para 30 dias e **declare isso no output**.

## Plausibilidade de valor (`G03`)

| Sinal | Limiar | Interpretação |
|---|---|---|
| Valor médio por conversão vs ticket médio informado | fora de **±40%** | Alerta |
| Valor idêntico em ≥ **95%** das conversões | — | Violação: tag sem `value` dinâmico |
| `always_use_default_value = TRUE` numa conversão de receita | — | Violação |
| Valor médio por conversão | **≤ R$ 5** numa conta de e-commerce | Violação (o caso "compra de R$ 1") |

Se o ticket médio não for conhecido, **pergunte**. Sem essa referência, os dois primeiros sinais são
inúteis e devem ser marcados como `NAO_VERIFICADO` — não chutados.

## Janelas

| Janela | Valor | Uso |
|---|---|---|
| Medição | 30 dias, terminando **3 dias atrás** | Discrepância, parametrização, ponderação por gasto |
| Vitalidade | 7 dias corridos | "Chegou alguma coisa?" |

O corte de 3 dias na janela de medição **não é negociável**. Sem ele, atribuição retroativa gera
discrepância falsa em conta saudável — a causa nº 1 de falso positivo neste tipo de auditoria.
