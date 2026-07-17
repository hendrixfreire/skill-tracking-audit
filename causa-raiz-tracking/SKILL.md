---
name: causa-raiz-tracking
description: Investigação de causa raiz de problemas de tracking — inventaria o container GTM publicado via API, observa as tags disparando de verdade na rede do navegador via extensão do Chrome, e cruza os dois para achar duplo disparo, pixel hardcoded, valor errado no dataLayer e falta de event_id para dedup de CAPI. Use quando a skill integridade-coleta apontou um sintoma que precisa de confirmação (discrepância, suspeita de duplo disparo), ou quando o analista precisa provar POR QUE a coleta está errada, não só QUE está errada.
---

# Causa Raiz de Tracking

A skill `integridade-coleta` mede sintomas: discrepância, valor implausível, tag morta. Esta skill
faz a etapa seguinte — **pega o disparo em flagrante**. Ela responde perguntas que número nenhum de
API de relatório responde:

- A tag dispara **duas vezes** no mesmo evento?
- Existe pixel **hardcoded no site**, fora do GTM, duplicando o que o container já faz?
- O dataLayer está empurrando o **valor certo** — ou `1` fixo?
- Os eventos do navegador carregam **`event_id`** para a CAPI deduplicar?

## Relação com a integridade-coleta

Esta é a skill cara. A `integridade-coleta` roda em minutos só com credencial de API; esta exige
acesso de leitura ao GTM do cliente, permissão de site na extensão do Chrome e um navegador aberto.
**Não rode as duas juntas por padrão.** O fluxo normal é:

1. `integridade-coleta` aponta: "Meta +47% sobre o GA4, compatível com falha de dedup — verifique".
2. Esta skill recebe essa hipótese, vai ao container e à rede, e volta com:
   **CONFIRMADO** (com a evidência), **REFUTADO** (com a evidência) ou **INCONCLUSIVO** (com o que
   faltou para decidir).

Ela também roda standalone, sem hipótese prévia — vira uma varredura completa de disparo. Mas se
existir um relatório da `integridade-coleta`, **peça-o primeiro**: investigar com hipótese é mais
rápido e mais barato que varrer tudo.

## O que esta skill NÃO faz

- **Não compra.** Ver a tag de Purchase disparar exige uma compra real. O agente nunca insere dados
  de pagamento. Purchase só é verificável pelo protocolo da seção "Purchase" abaixo — pedido de
  teste feito pelo cliente ou ambiente de staging. Sem isso, Purchase fica `NAO_VERIFICADO` e os
  eventos de funil servem de proxy declarado.
- **Não edita o container.** Escopo `tagmanager.readonly`, sempre. Recomendação de mudança vai para
  o output; a mudança em si é do analista ou do dev.
- **Não usa computer use para pilotar a interface do GTM.** O container se lê pela API numa chamada.
  Clicar na SPA do GTM é lento, frágil e entrega o mesmo dado.
- **Não navega além do necessário.** O roteiro é fixo (home → produto → carrinho → checkout). Nada
  de explorar o site do cliente livremente.

## Pré-requisitos

| O quê | Como |
|---|---|
| GTM leitura | Conta do analista com acesso ao container, OAuth com escopo `tagmanager.readonly` |
| IDs do GTM | `accountId` e `containerId` (o público `GTM-XXXXXX` está no fonte do site) |
| Extensão Chrome | Conectada, com permissão para o domínio do cliente |
| URL do site | Home + uma URL de produto (peça ao analista se a home não levar a produto em 1 clique) |
| Do relatório v1 | Se existir: as hipóteses em aberto e os ids dos achados (`X02`, `G03`...) |

Se o GTM não estiver acessível mas o navegador estiver, rode só a fase de rede e declare a cobertura
parcial. O inverso também vale. **Fonte faltando reduz cobertura, não aborta a skill** — mesma regra
da v1.

## Fase 1 — Inventário do container (API)

Chamadas em `references/gtm-api.md`. Leia **sempre a versão publicada** (`versions/live`), nunca o
workspace — workspace é rascunho e mente sobre o que está no ar.

| id | Check | Tipo |
|---|---|---|
| `C01` | Existe tag para cada plataforma ativa da conta (GA4 config, Google Ads conversion, Meta Pixel) na versão publicada | Invariante |
| `C02` | Nenhum evento tem duas tags equivalentes no container (dois Meta Pixel base, duas tags de conversão iguais) | Invariante |
| `C03` | As tags de conversão leem valor de variável do dataLayer — não valor fixo, não vazio | Invariante |
| `C04` | Triggers das tags de conversão são específicos (evento de compra), não genéricos (All Pages) | Invariante |
| `C05` | O workspace não tem mudanças relevantes não publicadas há muito tempo | Diretriz |

O resultado da fase 1 é um **inventário**: lista de tags com trigger e variáveis, que a fase 3 vai
cruzar contra o que a rede mostrar. Guarde-o estruturado.

## Fase 2 — Observação na rede (extensão do Chrome)

Protocolo detalhado em `references/navegacao.md`. Resumo: carregar home → produto → adicionar ao
carrinho → iniciar checkout, lendo `read_network_requests` a cada passo e o fonte da página na home.

| id | Check | Tipo |
|---|---|---|
| `N01` | Cada evento observável dispara **exatamente uma vez** por plataforma (PageView, ViewContent, AddToCart, InitiateCheckout) | Invariante |
| `N02` | Os disparos do Meta carregam `event_id` (o parâmetro `eid`) — sem ele a CAPI não tem como deduplicar | Invariante |
| `N03` | O payload dos eventos de valor carrega `value` e `currency` plausíveis — produto de R$ 340 mandando `value=340`, não `value=1` nem vazio | Invariante |
| `N04` | Não há pixel hardcoded no fonte da página que também exista no container (fonte contém `fbq('init')`/`gtag('config')` E o container tem a tag equivalente) | Invariante |
| `N05` | Consent mode / bloqueio de consentimento não está suprimindo os disparos silenciosamente (nenhuma requisição sai antes de aceitar o banner — teste aceitar e recarregar) | Diretriz |

Dois cuidados que valem mais que o protocolo inteiro:

- **`N01` é o flagrante.** Duas requisições `facebook.com/tr` com o mesmo `ev` no mesmo pageview é
  duplo disparo provado — a evidência que a v1 só conseguia inferir por discrepância. Capture as duas
  URLs completas: elas são a evidência do output.
- **`N04` é a causa raiz clássica do `N01`.** Pixel colado no tema da loja + pixel no GTM. O
  container parece impecável e a conta conta tudo em dobro. Só o cruzamento fonte-da-página ×
  container pega isso.

## Fase 3 — Cruzamento

| id | Check | Tipo |
|---|---|---|
| `R01` | Tudo que o container declara para as páginas navegadas apareceu na rede | Invariante |
| `R02` | Tudo que apareceu na rede existe no container — o que sobra é hardcoded ou de plugin/tema | Invariante |

`R01` violado = tag configurada que não dispara (trigger errado, consentimento, erro de JS).
`R02` violado = disparo órfão — some com o `N04` para nomear a origem provável.

## Purchase — o evento que importa e que não dá para forçar

Nenhuma navegação chega ao Purchase sem pagar. As opções, em ordem de preferência:

1. **Pedido de teste pelo cliente** — o analista combina um pedido real (pode ser cupom de 100%),
   avisa o horário, e a skill observa a rede durante o checkout OU confere o evento no lado do
   servidor depois. É o único teste completo.
2. **Staging/sandbox** — se a loja tiver, rodar o funil inteiro lá. Vale menos: staging nem sempre
   espelha o tracking de produção, declare isso.
3. **Proxy declarado** — sem 1 nem 2, o veredito sobre Purchase é herdado do funil: "AddToCart e
   InitiateCheckout disparam limpos e com event_id; **é provável** que Purchase também, **não
   verificado diretamente**". Nunca escreva que Purchase está ok sem tê-lo visto.

## Output

Mesmo formato da v1 — bloco JSON estruturado + prosa com recomendação primeiro — com duas diferenças:

1. **Sem score.** Esta skill não mede saúde, ela julga hipóteses. O campo central é `hipoteses`:

```json
{
  "conta": "<cliente>",
  "hipoteses": [
    {
      "origem": "X02",
      "hipotese": "Falha de dedup CAPI explicando +47% de receita no Meta",
      "veredito": "CONFIRMADO",
      "evidencia": "Eventos do navegador disparam sem event_id (parâmetro eid ausente em 4/4 eventos observados). A CAPI não tem chave para deduplicar: cada conversão chega duas vezes.",
      "acao": "Dev: gerar event_id no navegador e enviar o mesmo id no evento de servidor.",
      "responsavel": "desenvolvedor"
    }
  ],
  "achados_novos": [ { "id": "N04", "...": "..." } ],
  "cobertura": {
    "gtm": true, "navegador": true,
    "eventos_observados": ["PageView", "ViewContent", "AddToCart", "InitiateCheckout"],
    "purchase": "NAO_VERIFICADO — proxy pelo funil"
  }
}
```

2. **Todo achado nomeia o responsável**: `analista` (resolve na plataforma/GTM), `desenvolvedor`
   (mexe em código/dataLayer) ou `cliente` (decisão ou acesso). Esta skill vive na fronteira entre
   os três, e um achado sem dono é um achado que ninguém executa.

Na prosa, evidência é URL de requisição, nome de tag do container, trecho de payload — coisa que o
dev possa conferir sem refazer a investigação. "A tag dispara duas vezes" sem as duas URLs é a v1 de
novo, mais cara.

## Guias

- **Hipótese primeiro, varredura depois.** Com relatório da v1 em mãos, ataque as hipóteses dele
  antes de qualquer coisa; a varredura completa é o modo standalone, não o padrão.
- **Veredito sobre hipótese é trinário** — CONFIRMADO, REFUTADO, INCONCLUSIVO — e os três exigem
  evidência. REFUTADO com prova ("dedup ok: eid presente e consistente; a discrepância vem de
  view-through") vale tanto quanto CONFIRMADO: poupa o dev de caçar um bug que não existe.
- **Não confunda os pixels dos outros.** Sites carregam TikTok, Pinterest, Hotjar, RD Station.
  Requisição que não é Google/Meta/GA4 não entra em `R02` como achado — no máximo uma linha de
  inventário.
- **Se o site tiver banner de consentimento, teste os dois estados** (antes e depois de aceitar) —
  senão `N05` e metade dos falsos "tag não dispara" ficam indistinguíveis.
- **Duas tentativas e para.** Se a extensão falhar ou o site bloquear a navegação duas vezes,
  reporte o que foi coletado com a cobertura honesta e devolva ao analista. Insistir queima tempo e
  não muda o resultado.
