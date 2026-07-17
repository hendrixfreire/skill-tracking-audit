# GTM — Tag Manager API v2

Leitura do container via REST. Escopo OAuth: `https://www.googleapis.com/auth/tagmanager.readonly`.
Base: `https://tagmanager.googleapis.com/tagmanager/v2`.

**Regra da v1 vale aqui:** se um endpoint ou campo não existir como descrito, `NAO_VERIFICADO` e
diga o que faltou. Não improvise.

## Descobrir os IDs

O ID público (`GTM-XXXXXX`) está no fonte do site. Para chegar ao caminho da API:

```
GET /accounts                                  # contas visíveis à credencial
GET /accounts/{accountId}/containers           # ache o container cujo publicId = GTM-XXXXXX
```

## Ler a versão publicada — o único endpoint que importa

```
GET /accounts/{accountId}/containers/{containerId}/versions:live
```

Retorna a versão **no ar**, com `tag[]`, `trigger[]`, `variable[]`, `builtInVariable[]` embutidos —
uma chamada só resolve a fase 1 inteira.

**Nunca leia o workspace para auditar** (`/workspaces/{id}/tags`): workspace é rascunho. A única
razão para tocá-lo é o check `C05` (mudanças não publicadas) — compare o `fingerprint`/lista de tags
do workspace default contra a versão live e reporte a diferença como diretriz.

## Como ler o retorno

### Tags (`tag[]`)

| Campo | Uso |
|---|---|
| `type` | Tipo da tag — ver tabela abaixo |
| `name` | Nome dado pelo publicador — vai na evidência |
| `firingTriggerId[]` | Cruzar com `trigger[]` para saber quando dispara |
| `parameter[]` | Onde vivem IDs de conversão, event name, valor |
| `paused` | `true` = ignorar para efeito de disparo, mas listar no inventário |

Tipos que interessam (campo `type`):

| `type` | O que é |
|---|---|
| `gaawc` | GA4 Configuration |
| `gaawe` | GA4 Event |
| `awct` | Google Ads Conversion Tracking |
| `sp` | Google Ads Remarketing |
| `html` | **HTML customizado** — é aqui que pixels do Meta vivem no GTM |
| `img` | Pixel de imagem |

**O Meta Pixel não tem tipo nativo**: ele aparece como `html` com `fbq('init', ...)` /
`fbq('track', ...)` dentro do `parameter` de nome `html`. Para `C02`, faça grep do conteúdo HTML das
tags por `fbq(` — duas tags com `fbq('init')` para o mesmo pixel ID é duplicação dentro do próprio
container.

### Triggers (`trigger[]`)

| `type` | Leitura para `C04` |
|---|---|
| `pageview`, `domReady`, `windowLoaded` | Genérico — correto para tag base, **errado** para tag de conversão |
| `customEvent` | Específico — o esperado para conversão (ex.: evento `purchase` do dataLayer) |

Tag de conversão de compra (`awct` ou `html` com `fbq('track','Purchase')`) pendurada em trigger
`pageview` de All Pages = `C04` violado: dispara em toda página, não na compra.

### Variáveis (`variable[]`)

Para `C03`: siga o `parameter` de valor da tag de conversão (`conversionValue` no `awct`; o campo
`value` dentro do `fbq('track','Purchase', {...})` no `html`). Ele deve referenciar `{{variável}}`
do tipo `v` (Data Layer Variable). Valor literal fixo (`1`, `0`, string vazia) = violação — é a
face-container do caso "compra de R$ 1", e cruza com o `G03`/`N03`.

## Chamada de exemplo

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://tagmanager.googleapis.com/tagmanager/v2/accounts/$ACC/containers/$CTR/versions:live" \
  | jq '{tags: [.tag[] | {name, type, paused, firingTriggerId}],
         triggers: [.trigger[] | {triggerId, name, type}]}'
```

O token vem do fluxo OAuth do analista (a conta dele já tem acesso ao container do cliente). Se não
houver token utilizável, a fase 1 inteira é `NAO_VERIFICADO` — a fase 2 (rede) não depende dela.
