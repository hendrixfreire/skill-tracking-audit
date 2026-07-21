# Catálogo de checks — C · R · U

Cada check tem **id**, o que olha, como se responde e o critério de veredito. É o que torna a
auditoria repetível entre contas: o mesmo id significa a mesma coisa em todo relatório.

## Vocabulário

| Tipo | Significado |
|---|---|
| **Inv** (invariante) | Deveria ser sempre verdade. Violação é achado. |
| **Dir** (diretriz) | Depende do contexto. Violação é pergunta, não veredito. |
| **Sinal** | Não julga — aponta coincidência para investigação. |

| Veredito | Quando |
|---|---|
| `PASSOU` | verificado e correto |
| `FALHOU` | verificado e incorreto — vira achado |
| `NAO_VERIFICADO` | **não foi possível verificar**, sempre com o motivo |
| `NAO_APLICAVEL` | o check não faz sentido nesta conta (ex.: `C04` sem Google Ads) |

**Nunca `PASSOU` no que não foi lido.** Ausência de evidência é `NAO_VERIFICADO`, não aprovação.

Convenção dos comandos abaixo: defaults já fixados (`gtm config set default*`) e
`outputFormat json`. Onde aparece `LIVE`, é o resultado de:

```bash
gtm versions live -o json > /tmp/live.json
```

---

# Fase 2 — Configuração (C01–C10)

O que o container **declara**. Config prova intenção, nunca comportamento.

---

### `C01` — Cobertura de plataformas · Inv

Existe Google tag/GA4 config em All Pages, tag de evento `purchase` do GA4, conversão do Google Ads
e pixel Meta (base + eventos), para **cada plataforma com gasto na conta**.

```bash
# comece pelo censo de tipos — é o mapa da conta em uma linha
jq -r '[.tag[] | .type] | group_by(.) | map({(.[0]): length}) | add' /tmp/live.json
jq '[.tag[] | {name, type, paused, firingTriggerId}]' /tmp/live.json
# pixels dentro de tags html
jq -r '[.tag[] | select(.type=="html") | .parameter[]? | select(.key=="html") | .value] | .[]' /tmp/live.json | grep -o "fbq('[a-z]*'" | sort | uniq -c
```

Tipos que respondem este check: `googtag` (Google tag / GA4 config atual) ou `gaawc` (legado),
`gaawe` (evento GA4), `awct` (conversão Ads), `gclidw` (Conversion Linker), `html` ou `cvt_…`
(pixel Meta e demais). Ver o mapa completo em [`gtm-cli.md`](gtm-cli.md#6-mapa-de-tipos).

**FALHOU** se plataforma com gasto não tem tag no container. Antes de fechar o veredito: a tag pode
viver fora do GTM (hardcode no tema) — isso é `NAO_VERIFICADO` aqui e vira predição para `R01`/`R03`.

---

### `C02` — Fiação do purchase · Inv

O gatilho do purchase escuta o evento certo do dataLayer, e `value`, `currency`,
`items`/`transaction_id` vêm de **variáveis do dataLayer**, não de valor fixo.

```bash
# tags de conversão e seus parâmetros de valor
jq '[.tag[] | select(.type=="awct" or .type=="gaawe") | {name, type, parameter}]' /tmp/live.json
# variáveis do tipo data layer
jq '[.variable[] | select(.type=="v") | {name, parameter}]' /tmp/live.json
```

**FALHOU** se o valor é literal (`1`, `0`, vazio) em vez de `{{variável}}`. É a origem, dentro do
container, do sintoma "toda compra vale R$ 1". Correção em [`correcoes.md`](correcoes.md#1).

---

### `C03` — `transaction_id` presente · Inv

No GA4 (dedup de transação) e na conversão do Ads (dedup de conversão).

```bash
# Google Ads: a chave é orderId
jq -r '.tag[] | select(.type=="awct") | .name as $n | (.parameter[]? | select(.key=="orderId") | .value) as $v | "\($n): orderId=\($v // "AUSENTE")"' /tmp/live.json
# GA4: transaction_id dentro do mapa de parâmetros do evento
jq -r '.tag[] | select(.type=="gaawe") | .name as $n | (.parameter|tostring) as $p | select($p|test("transaction_id")) | $n' /tmp/live.json
```

⚠️ **Procure pela chave, não pelo valor.** Na conversão do Ads o parâmetro chama-se `orderId` e o
valor é o nome da variável do cliente — que pode ser qualquer coisa (`{{oderFormTrasactionId}}`,
com typo e tudo). Filtrar `.value` por `"transaction_id"` produz **falso negativo**: reporta
ausente numa tag que tem, e gera uma correção desnecessária num container que já estava certo.

**FALHOU** se a chave não existe ou aponta para variável vazia. Sem ele, refresh na página de
obrigado reconta a compra — explica receita inflada sem duplo disparo de tag.

---

### `C04` — Conversion Linker · Inv · `NAO_APLICAVEL` sem Google Ads

Tag presente **e em All Pages** sempre que houver conversão do Ads no container.

```bash
jq '[.tag[] | select(.type=="gclidw") | {name, type, firingTriggerId, paused}]' /tmp/live.json
```

**O tipo é `gclidw`** — não `sp`. `sp` é Google Ads Remarketing, outra coisa. Buscar pelo nome
("Linker", "Vinculador") é frágil: o nome é livre e muda por idioma.

**FALHOU** se ausente ou se o gatilho não é All Pages. Sem ela o `gclid` não vira cookie e a
conversão degrada silenciosamente.

---

### `C05` — Duplicidade interna · Inv

Duas tags da mesma plataforma no mesmo gatilho — dois GA4 config, dois pixels base.

Tags **nativas** agrupam por tipo + gatilho:

```bash
jq -r '.tag[] | select(.paused != true and .type != "html") | "\(.type)\t\(.firingTriggerId|tostring)\t\(.name)"' /tmp/live.json | sort | awk -F'\t' '{k=$1"|"$2; c[k]++; n[k]=n[k]" · "$3} END{for(i in c) if(c[i]>1) print c[i]"x " i n[i]}'
```

Tags **`html` exigem desambiguação por conteúdo** — `html` é tipo genérico, e agrupá-lo por
tipo+gatilho produz falso positivo em massa (Clarity, TikTok, chat e pixel Meta convivem em All
Pages sem que nada esteja duplicado):

```bash
jq -r '.tag[] | select(.paused != true and .type=="html") | .name as $n | (.parameter[]? | select(.key=="html") | .value) as $h | [$n, (if ($h|test("fbq\\(.init")) then "meta-init" elif ($h|test("gtag\\(.config")) then "gtag-config" else "outro" end)] | @tsv' /tmp/live.json | awk -F'\t' '$2!="outro"{c[$2]++; n[$2]=n[$2]" · "$1} END{for(i in c) print c[i]"x "i n[i]}'
```

**FALHOU** com duas tags equivalentes ativas no mesmo gatilho — mesma plataforma, não só mesmo
tipo. A duplicidade **container × hardcode no site** é invisível aqui: só aparece cruzando com
`R01`/`R03`.

---

### `C06` — Tags pausadas relevantes · Dir

Tag de plataforma com gasto, pausada.

```bash
jq '[.tag[] | select(.paused==true) | {name, type}]' /tmp/live.json
```

Pode ser intencional. **Vira pergunta ao cliente, não veredito.**

---

### `C07` — Consent mode como mecânica · Dir

Defaults por região, tags condicionadas a consentimento, comportamento antes da escolha.

```bash
# distribuição — comece por aqui
jq -r '[.tag[] | .consentSettings.consentStatus // "ausente"] | group_by(.) | map({(.[0]): length}) | add' /tmp/live.json
# só as que realmente condicionam disparo a consentimento
jq '[.tag[] | select(.consentSettings.consentStatus != null and .consentSettings.consentStatus != "notSet") | {name, consentSettings}]' /tmp/live.json
jq '[.tag[] | select(.name|test("consent|cookie|lgpd";"i")) | {name, type, firingTriggerId}]' /tmp/live.json
```

⚠️ **`consentSettings != null` não discrimina nada** — o campo existe em toda tag, quase sempre com
`consentStatus: "notSet"`. Filtre por `!= "notSet"`. Um container inteiro em `notSet` é um achado
em si: nenhuma tag está condicionada a consentimento no GTM (o bloqueio, se houver, vem da CMP).

Lido como **mecânica de disparo** (o que bloqueia o quê), não como conformidade jurídica. Consent
que segura o GA4 além do necessário é hipótese direta para taxa de sessões atribuídas baixa —
confirma-se em `R06`.

---

### `C08` — Gatilhos frágeis · Dir

Evento crítico pendurado em clique de elemento CSS ou visibilidade, em vez de evento de dataLayer.

```bash
jq '[.trigger[] | select(.type=="click" or .type=="linkClick" or .type=="elementVisibility") | {triggerId, name, type, filter}]' /tmp/live.json
```

⚠️ **Os tipos de gatilho são minúsculos na leitura e MAIÚSCULOS na escrita.** A API retorna
`pageview`, `customEvent`, `domReady`, `windowLoaded`, `click`, `linkClick`; o CLI aceita
`PAGEVIEW`, `CUSTOM_EVENT`, `CLICK` no `--type` do `create`. Filtrar leitura com a forma maiúscula
retorna vazio — e vazio lido como "não há gatilho frágil" é exatamente o `PASSOU` falso que esta
skill proíbe. Na dúvida, liste os tipos presentes antes de filtrar:
`jq -r '[.trigger[].type] | group_by(.) | map({(.[0]): length}) | add' /tmp/live.json`

Quebra com redesign, sem aviso. **FALHOU** quando o gatilho frágil sustenta evento de conversão ou
de e-commerce. Correção em [`correcoes.md`](correcoes.md#4).

---

### `C09` — Linha do tempo de versões · Sinal

Data da última publicação × datas dos degraus detectados na camada de números.

```bash
# data da última publicação — o fingerprint é epoch em MILISSEGUNDOS
jq -r '.fingerprint | tonumber / 1000 | strftime("%Y-%m-%d %H:%M")' /tmp/live.json
jq '{containerVersionId, name}' /tmp/live.json
# inventário de versões (SEM datas — ver aviso)
gtm version-headers list -o json | jq '[.[] | {containerVersionId, name, numTags, numTriggers}]'
# uma versão específica, com seu fingerprint
gtm versions get --version-id <id> -o json | jq '{containerVersionId, name, fingerprint}'
gtm workspaces status -o json      # mudanças pendentes, não publicadas
```

⚠️ **`version-headers` não traz data nenhuma** — só id, nome e contagens. A data sai do
`fingerprint` (epoch ms) de cada versão, o que exige um `versions get` por versão. Para o uso
típico do check basta a data da **live**; puxe versões anteriores só se houver um degrau específico
a datar.

Mudança publicada às vésperas de um degrau é causa **provável**. Aponte a coincidência; não afirme
causalidade. O `workspaces status` transforma o antigo carimbo "pode haver rascunho divergente" em
número: quantas mudanças pendentes, e quais.

---

### `C10` — dataLayer declarado × populado · handoff

O container mostra a variável; nunca o conteúdo dela.

**Sempre `NAO_VERIFICADO` nesta fase, por construção.** É o handoff para `R02`. Liste aqui as
variáveis `v` que a fase 3 precisa confirmar populadas:

```bash
jq -r '.variable[] | select(.type=="v") | .parameter[]? | select(.key=="name") | .value' /tmp/live.json
```

---

## Fecho da fase 2 — registrar as predições

Antes de abrir o navegador, escreva o que a configuração **prevê** que vai disparar em cada passo
do roteiro:

```
home              → GA4 page_view (tag "GA4 - Config") · Meta PageView (tag "Meta - Base")
página de produto → + GA4 view_item · Meta ViewContent
add to cart       → + GA4 add_to_cart · Meta AddToCart
checkout          → + GA4 begin_checkout · Meta InitiateCheckout
```

**Isto é obrigatório.** É o antídoto do viés de confirmação que a ordem config→runtime introduz, e
é o que transforma a fase 3 em teste em vez de passeio. Sem predição escrita, a fase 3 vira busca
por confirmação do que já se acredita.

---

# Fase 3 — Comportamento (R01–R07)

O que o site **faz**. Navegação instrumentada no roteiro
**home → produto → adicionar ao carrinho → carrinho → início do checkout**, capturando a cada passo
as chamadas de rede (`/g/collect` do GA4 com nome do evento; `googleadservices`/conversão do Ads;
`facebook.com/tr` do pixel com evento), o dataLayer e o estado de consentimento.

Cada observação é confrontada com a predição da fase 2. **Divergência tem nome:**

| Divergência | Significado |
|---|---|
| Predito, não observado | config que não dispara — gatilho errado, consent, erro de JS |
| Observado, não predito | **disparo órfão** — hardcode no tema, plugin, outro container |

O disparo órfão é a causa raiz clássica do duplo disparo, e só aparece nesse cruzamento: o
container parece impecável e a conta conta tudo em dobro.

---

### `R01` — GA4 único por pageview · Inv

Um `page_view` por página.

Dois hits = GA4 duplicado (GTM + hardcode). Sessões e receita inflam. Capture **as duas URLs
completas** — elas são a evidência.

---

### `R02` — Escada de e-commerce dispara · Inv

`view_item`, `add_to_cart`, `begin_checkout` chegando com objeto `ecommerce` populado (id, preço).

É aqui que `C10` se fecha: a variável declarada no container tem, ou não tem, conteúdo real.

---

### `R03` — Pixel Meta único e com eventos · Inv

Um `PageView` base por página; eventos padrão da escada disparando.

`tr` duplicado = evento dobrado. Verifique também o parâmetro de dedup (`eid`/`event_id`): sem ele,
a CAPI não tem chave para deduplicar e cada conversão chega duas vezes.

---

### `R04` — Cookies de atribuição · Inv

`_fbp`/`_fbc` (Meta) e `gclid` persistido (Ads, via Conversion Linker) presentes após aterrissar com
clique simulado: `?gclid=teste`, `?fbclid=teste`.

Fecha o par com `C04`: a tag existe no container **e** o cookie aparece no navegador.

---

### `R05` — UTM → sessão · Inv

Aterrissar com UTM e conferir que o `/g/collect` inicial carrega a origem.

Se o site redireciona e perde o parâmetro no caminho, está achada a causa de sessões caindo em
direct — e a correção **não é no GTM**, é no site (`responsavel: desenvolvedor`).

---

### `R06` — Consent na prática · Dir

O que dispara antes da escolha do banner, o que destrava depois do aceite. **Teste os dois
estados** — sem isso, `R06` e metade dos falsos "a tag não dispara" ficam indistinguíveis.

Eventos perdidos pré-consentimento são perda estrutural de sessão: **quantificar, não julgar**.

---

### `R07` — Ordem e latência de carregamento · Dir

GA4 disparando tarde (após interação/scroll, ou segundos depois do load) perde a sessão de quem sai
rápido. Causa silenciosa de sessões não atribuídas.

---

### Purchase — o evento que importa e que não se força

O roteiro **para no início do checkout**. Nenhuma navegação chega ao purchase sem pagar, e a skill
nunca insere dados de pagamento. Em ordem de preferência:

1. **Pedido de teste combinado com o cliente** — cupom de 100%, horário avisado, observando a rede
   durante o checkout. É o único teste completo.
2. **Staging/sandbox** — se existir. Vale menos: staging nem sempre espelha o tracking de produção.
   Declare a ressalva.
3. **Triangulação** — config correta (`C02`, `C03`) + evento chegando vivo com valor no
   Events Manager / GA4. O veredito é herdado: "a escada dispara limpa; **é provável** que purchase
   também; **não verificado diretamente**".

**Nunca escreva que purchase está ok sem tê-lo visto.** Afirmar "purchase testado no site" sem
transação é mentira metodológica.

---

# Fase 4 — UTMs por anúncio (U)

Varredura dos parâmetros de URL de **todos os anúncios ativos com gasto** de cada plataforma paga
(`url_tags` no Meta, final URLs/suffix no Google, equivalentes nas demais). O GA4 só enxerga o que
a URL carrega — esta varredura mede quanto de cada canal ele consegue ver.

| Estado | Sinal | Por quê |
|---|---|---|
| **UTM correta** | Verde | `utm_source`/`utm_medium` canônicos, medium puro, granularidade em `campaign`/`term`/`content`. O canal soma numa linha e cai no agrupamento certo. |
| **Fora das melhores práticas** | **Amarelo** | O número **existe**, só está desorganizado — medium poluído, variação livre de source, granularidade no campo errado. O tráfego fragmenta e escapa do agrupamento (Unassigned), mas é atribuível e **recuperável** por correção de padrão. |
| **UTM inexistente** | **Vermelho** | O GA4 fica **cego ao canal**: a sessão cai em direct/(none) e **nenhuma correção retroativa recupera**. Gasto sem visibilidade na fonte da verdade. |

Regras da varredura:

- Sai sempre em **dois níveis**: agregado (% do gasto do canal em cada estado) **e lista nominal**
  (id estável, nome, gasto, ativo hoje).
- O padrão correto proposto é **um por conta**, espelhado do melhor padrão já em uso nela — não um
  padrão teórico importado de fora.
- **Exceção Google:** com auto-tagging (`gclid`) ligado e são, UTM ausente não cega o GA4. O check
  vira "a UTM manual não **conflita** com o gclid".
- `responsavel: analista` — a correção é no gerenciador de anúncios, **não no container**. Sai como
  proposta, nunca como escrita do CLI.
