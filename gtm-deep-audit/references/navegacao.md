# Protocolo de navegação e leitura de rede

Fase 2 da skill `causa-raiz-tracking`, via extensão do Chrome (`mcp__claude-in-chrome__*`).
O roteiro é **fixo**: home → produto → adicionar ao carrinho → iniciar checkout. Nada além disso.

## Setup

1. `tabs_context_mcp` para ver o estado do navegador; crie aba nova com `tabs_create_mcp` — não
   reaproveite aba do usuário.
2. Se a extensão não tiver permissão para o domínio, **pare e peça ao analista** para conceder na
   extensão. Não contorne.
3. Se houver banner de consentimento: primeiro leia a rede **antes** de aceitar (estado 1), depois
   aceite e recarregue (estado 2). Os dois estados são dados — ver `N05`.

## Roteiro

| Passo | Ação | Eventos esperados |
|---|---|---|
| 1 | Carregar a home | PageView (Meta), `page_view` (GA4), tag base do Ads |
| 2 | Abrir uma página de produto | PageView + ViewContent (Meta), `view_item` (GA4) |
| 3 | Adicionar ao carrinho | AddToCart (Meta), `add_to_cart` (GA4) |
| 4 | Iniciar checkout | InitiateCheckout (Meta), `begin_checkout` (GA4) |

Em **cada passo**: aguarde o carregamento terminar, chame `read_network_requests` filtrando pelos
padrões abaixo, e registre as URLs completas encontradas **antes** do próximo passo — navegação
limpa o buffer mental de qual requisição pertence a qual página.

Na home, adicionalmente: `get_page_text`/fonte da página para o check de hardcoded (`N04`).

**Pare no passo 4.** Não preencha formulário de checkout, não insira dado pessoal nem de pagamento.

## Padrões de rede

| Plataforma | Padrão da requisição | Evento em |
|---|---|---|
| Meta Pixel | `facebook.com/tr` | parâmetro `ev` (`PageView`, `ViewContent`, `AddToCart`, `InitiateCheckout`, `Purchase`) |
| GA4 | `google-analytics.com/g/collect` ou `analytics.google.com/g/collect` | parâmetro `en` |
| Google Ads | `googleads.g.doubleclick.net/pagead/viewthroughconversion` ou `google.com/pagead/1p-conversion` | rótulo na URL |

Parâmetros do Meta que são evidência direta:

| Parâmetro | Check | Leitura |
|---|---|---|
| `ev` | `N01` | Nome do evento. **Duas requisições `tr` com mesmo `ev` no mesmo passo = duplo disparo provado.** Guarde as duas URLs inteiras. |
| `id` | — | Pixel ID. Dois `id` diferentes = dois pixels no site (às vezes legítimo — agência antiga — mas sempre reportar). |
| `eid` | `N02` | O event_id da dedup com CAPI. **Ausente = a CAPI não tem chave para deduplicar.** É a evidência que confirma ou refuta a hipótese `X02` da v1. |
| `cd[value]`, `cd[currency]` | `N03` | Valor e moeda do evento. `value` ausente ou `1` fixo num produto de preço conhecido = dataLayer empurrando valor errado. |

GA4: `en` é o evento; `ep.value`/`epn.value` e `cu` fazem o papel de valor e moeda para `N03`.

## Detecção de hardcoded (`N04`)

No fonte da home, procure:

```
fbq('init'          → Meta Pixel direto no tema
gtag('config'       → GA4/Ads fora do GTM
GTM-                → containers GTM (mais de um GTM-XXXXXX = dois containers!)
```

A lógica do cruzamento: script no fonte **fora** do snippet do GTM + tag equivalente **dentro** do
container (inventário da fase 1) = mesma tag instalada duas vezes = causa raiz do duplo disparo.
Atenção ao falso positivo: o próprio GTM injeta `fbq`/`gtag` em runtime. O que denuncia hardcoded é
o script estar no **HTML-fonte** (view-source), não no DOM renderizado.

Plugins de plataforma (Shopify: "Facebook & Instagram app"; Woo: pixels de plugin) são a origem mais
comum de hardcoded que ninguém lembra que instalou — se houver duplo disparo e nada no fonte, o
palpite seguinte é app da plataforma injetando server-side ou via sandbox próprio; nomeie a hipótese
no output com responsável `cliente`.

## Interpretação por combinação

| Fase 1 (container) | Fase 2 (rede) | Conclusão |
|---|---|---|
| Tag existe | Dispara 1x, com `eid` e valor certo | Saudável — diga isso no output |
| Tag existe | Dispara 2x | `N01` + provável `N04`: procure a segunda origem no fonte |
| Tag existe | Não dispara | `R01`: trigger errado, consentimento (`N05`) ou erro JS |
| Tag não existe | Dispara | `R02`: hardcoded ou plugin — inventarie a origem |
| Tag existe | Dispara sem `eid` | `N02`: dedup CAPI sem chave — responsável `desenvolvedor` |
| Tag existe | Dispara com `value=1` | `N03`: dataLayer errado — cruza com `G03` da v1 |

## Limites

- **Máximo 2 tentativas** por passo que falhar (página não carrega, elemento não clica). Depois,
  registre a cobertura parcial e siga para o que der.
- **Sites com login/paywall**: peça credencial de teste ao analista ou declare o funil inacessível.
  Não crie conta no site do cliente por conta própria.
- **Popups/alerts**: não acione elementos que abram dialog nativo do navegador (confirm/alert) —
  travam a sessão da extensão.
