# Correção — protocolo de escrita e receitas

A fase que separa esta skill das anteriores da família: além de apontar a causa, ela **aplica a
correção no container** — em workspace isolado, deixando a versão pronta para revisão humana.

**A skill nunca publica.** `gtm versions publish` está fora de escopo, sem exceção. O que ela
entrega é uma versão criada, um diff transcrito e o roteiro de teste em Preview. Quem publica é o
humano com acesso, depois de testar.

---

## Parte 1 — O protocolo

Sete passos, todos obrigatórios, nesta ordem. Pular um invalida os seguintes.

### Passo 1 — Backup, antes de tocar em qualquer coisa

```bash
D=$(date +%Y-%m-%d)
gtm versions live -o json  > "diagnostico/${D}-backup-live.json"
gtm tags list      -o json > "diagnostico/${D}-backup-tags.json"
gtm triggers list  -o json > "diagnostico/${D}-backup-triggers.json"
gtm variables list -o json > "diagnostico/${D}-backup-variables.json"
```

Na pasta da conta. O caminho vai no JSON de saída, campo `execucao.backup`.

### Passo 2 — Workspace isolado

```bash
gtm workspaces create --name "auditoria-${D}" \
  --description "Correções da auditoria de $D — ver relatório" -o json
gtm config set defaultWorkspaceId <id-retornado>
```

**Nunca o `Default`. Nunca um workspace com mudanças de outra pessoa** — verifique antes:

```bash
gtm workspaces list -o json
```

Workspace alheio com mudanças pendentes significa que alguém está trabalhando ali; misturar as
correções torna impossível separar quem mudou o quê na hora de publicar.

### Passo 3 — Sync antes de escrever

```bash
gtm workspaces sync -o json
```

**Conflito com a live é parada.** Reporte e devolva ao humano — resolver conflito no escuro é como
a auditoria vira incidente.

### Passo 4 — Inspecionar antes de criar

```bash
gtm tags get --tag-id <id-de-uma-equivalente-que-funciona> -o json
```

Herde o formato exato de `type` e `parameter` de algo que já funciona neste container. Obrigatório
para tags de template customizado, onde o `type` (`cvt_{containerId}_{templateId}`) só se descobre
olhando.

### Passo 5 — Aplicar

Uma correção por vez, cada uma amarrada ao achado que a justifica. JSON sempre em arquivo temporário
(ver [`gtm-cli.md`](gtm-cli.md#8-padrões-de-execução)). Paralelize só o que é independente.

**O que pode ser aplicado:** apenas achados com `responsavel: analista` — o que se resolve dentro
do container. Achado de `desenvolvedor` (dataLayer, código do site) ou de `cliente` (acesso,
decisão) sai como **proposta**, com `status_execucao: proposta`.

**O que exige confirmação no chat, item a item:** qualquer `delete`. O padrão para tag defeituosa é
**pausar ou corrigir**, nunca apagar — apagar destrói o histórico de configuração e não é
reversível pelo `revert`.

### Passo 6 — Diff

```bash
gtm workspaces status -o json
gtm workspaces preview -o json
```

Transcreva no relatório como **lista literal** do que mudou. Se o diff mostrar algo que a auditoria
não pediu, pare: ou o passo 2 pegou workspace errado, ou uma correção teve efeito colateral.

### Passo 7 — Criar a versão, e parar

```bash
gtm versions create \
  --name "auditoria ${D}" \
  --notes "1. valor de purchase agora lê o dataLayer (achado C02)
2. Conversion Linker criada em All Pages (achado C04)
3. GA4 config duplicado pausado (achado C05)" \
  -o json
```

As notas são o registro que sobrevive à conversa — quem abrir o container em seis meses precisa
entender por que a versão existe. Depois disso, **PARE**. Entregue:

- o `versionId` criado;
- o roteiro de teste em Preview;
- a recomendação de janela de publicação (baixo tráfego).

### Desfazer

```bash
gtm tags revert --tag-id <id>              # desfaz dentro do workspace
gtm triggers revert --trigger-id <id>
gtm variables revert --variable-id <id>
```

Apagar o workspace de auditoria devolve o container ao estado inicial. **Nada do que a skill fez
afeta o site até alguém publicar.**

### Roteiro de teste a entregar ao humano

1. GTM → **Preview**, escolhendo o workspace `auditoria-AAAA-MM-DD`.
2. Percorrer home → produto → adicionar ao carrinho → início do checkout.
3. Conferir, no Tag Assistant, que cada tag corrigida dispara **uma vez**, no momento certo, com o
   valor vindo do dataLayer.
4. Se houver pedido de teste combinado, conferir o purchase com valor e `transaction_id`.
5. Publicar em janela de baixo tráfego.
6. Rodar de novo a camada de números (`integridade-coleta`) em 7 dias para confirmar que o sintoma
   fechou.

---

## Parte 2 — Receitas

Cada receita: o achado que a dispara, o comando, o efeito esperado e o que testar. Os ids de tag,
gatilho e variável são de exemplo — leia os reais antes.

---

### 1. Valor de purchase hardcoded → variável do dataLayer

**Achado:** `C02` · **Responsável:** analista · **Sintoma que fecha:** receita achatada, toda compra
valendo o mesmo.

Se as variáveis do dataLayer ainda não existem, crie-as primeiro:

```bash
for v in value currency transaction_id; do
cat > /tmp/dlv_${v}.json << EOF
{"parameter":[
  {"type":"INTEGER","key":"dataLayerVersion","value":"2"},
  {"type":"BOOLEAN","key":"setDefaultValue","value":"false"},
  {"type":"TEMPLATE","key":"name","value":"ecommerce.${v}"}
]}
EOF
gtm variables create --name "DLV - ecommerce.${v}" --type v --config "$(cat /tmp/dlv_${v}.json)" &
done
wait
```

Depois aponte a tag de conversão para elas. O `--config` do `update` é **mesclado** com o existente:

```bash
cat > /tmp/fix_valor.json << 'EOF'
{"parameter":[
  {"type":"TEMPLATE","key":"conversionValue","value":"{{DLV - ecommerce.value}}"},
  {"type":"TEMPLATE","key":"currencyCode","value":"{{DLV - ecommerce.currency}}"}
]}
EOF
gtm tags update --tag-id <id> --config "$(cat /tmp/fix_valor.json)" -o json
```

**Efeito:** cada conversão passa a chegar com o valor real da compra.
**Testar:** no Preview, uma compra de valor conhecido chega com esse valor, não com `1`.

---

### 2. `transaction_id` ausente

**Achado:** `C03` · **Responsável:** analista (se a variável existe) ou **desenvolvedor** (se o
dataLayer não empurra o id) · **Sintoma que fecha:** compra recontada em refresh da página de
obrigado.

```bash
cat > /tmp/fix_txid.json << 'EOF'
{"parameter":[
  {"type":"TEMPLATE","key":"orderId","value":"{{DLV - ecommerce.transaction_id}}"}
]}
EOF
gtm tags update --tag-id <id-da-conversao-ads> --config "$(cat /tmp/fix_txid.json)" -o json
```

Na tag de evento do GA4, o parâmetro é `transaction_id` dentro do mapa de parâmetros do evento —
leia a estrutura da tag com `gtm tags get` antes de montar o config.

**Antes de aplicar, confirme que o dataLayer realmente empurra o id** (`R02`). Se não empurra, a
correção é no site: `status_execucao: proposta`, `responsavel: desenvolvedor`. Apontar uma tag para
uma variável vazia troca um defeito por outro pior — silencioso.

---

### 3. Conversion Linker ausente

**Achado:** `C04` · **Responsável:** analista · **Sintoma que fecha:** conversões do Ads degradando,
`gclid` não persistido.

```bash
ALL_PAGES=$(gtm triggers list -o json | jq -r '.[] | select(.type=="pageview") | .triggerId' | head -1)

gtm tags create \
  --name "Conversion Linker" \
  --type gclidw \
  --firing-trigger-id "$ALL_PAGES" \
  --config '{"parameter":[{"type":"BOOLEAN","key":"enableCookieOverrides","value":"false"}]}' \
  -o json
```

**O tipo é `gclidw`.** `sp` é Google Ads Remarketing — criar `sp` aqui produz uma tag de
remarketing com nome de Linker, que não persiste `gclid` nenhum e ainda parece resolvido no
inventário. Note também que a leitura filtra `pageview` minúsculo enquanto a criação usa `PAGEVIEW`
maiúsculo (ver [`gtm-cli.md`](gtm-cli.md#gatilhos)).

Se o gatilho All Pages não existir, crie antes:
`gtm triggers create --name "All Pages" --type PAGEVIEW --config '{}'`.

**Efeito:** o `gclid` da URL vira cookie primário e sobrevive à navegação até a compra.
**Testar:** aterrissar com `?gclid=teste` e conferir o cookie `_gcl_aw` (`R04`).

---

### 4. Gatilho frágil → evento de dataLayer

**Achado:** `C08` · **Responsável:** analista se o evento já existe no dataLayer; **desenvolvedor**
se for preciso criá-lo · **Sintoma que fecha:** evento que parou de disparar depois de um redesign.

```bash
cat > /tmp/ce_purchase.json << 'EOF'
{"customEventFilter":[{
  "type":"EQUALS",
  "parameter":[
    {"type":"TEMPLATE","key":"arg0","value":"{{_event}}"},
    {"type":"TEMPLATE","key":"arg1","value":"purchase"}
  ]
}]}
EOF
NEW=$(gtm triggers create --name "CE - purchase" --type CUSTOM_EVENT \
      --config "$(cat /tmp/ce_purchase.json)" -o json | jq -r '.triggerId')

gtm tags update --tag-id <id-da-tag> --firing-trigger-id "$NEW" -o json
```

`arg0` **precisa** ser `{{_event}}` — é o erro mais comum do CLI.

**Só troque o gatilho depois de confirmar em `R02` que o evento existe no dataLayer.** Trocar um
gatilho frágil que funciona por um evento que não é empurrado transforma uma tag instável em uma
tag morta.

**Testar:** no Preview, a tag dispara no evento — não no clique do elemento.

---

### 5. Tag duplicada → pausar a excedente

**Achado:** `C05` (config) ou `R01`/`R03` (runtime) · **Responsável:** analista · **Sintoma que
fecha:** sessões e receita infladas, evento contado duas vezes.

```bash
gtm tags update --tag-id <id-da-excedente> --paused true -o json
```

**Pausar, não apagar.** Pausar é reversível em um comando e preserva a configuração para quem
investigar depois. Se a duplicidade é **container × hardcode no site** (`R01`/`R03` mostraram um
disparo órfão), a tag do container pode estar certa e o hardcode ser o problema — nesse caso a
correção é remover o script do tema: `status_execucao: proposta`, `responsavel: desenvolvedor`.

Antes de pausar, decida qual das duas é a boa: a que tem a configuração mais completa (valor do
dataLayer, `transaction_id`, consent settings), não a mais antiga.

**Testar:** no Preview, um único disparo por evento.

---

### 6. Tag de plataforma ausente

**Achado:** `C01` · **Responsável:** analista.

Não há receita genérica: cada plataforma tem seu tipo e seus parâmetros. O caminho é sempre o
mesmo — **inspecionar antes de criar** (passo 4), usando como molde uma tag equivalente que já
funcione neste container ou noutro da mesma conta:

```bash
gtm tags get --tag-id <molde> -o json > /tmp/molde.json
jq '{parameter}' /tmp/molde.json > /tmp/nova.json     # ajuste os valores
gtm tags create --name "<nome>" --type "$(jq -r .type /tmp/molde.json)" \
  --firing-trigger-id <id> --config "$(cat /tmp/nova.json)" -o json
```

Antes: confirme em `R01`/`R03` que a tag não existe **fora** do GTM. Criar no container o que já
está hardcoded no tema produz duplicação — troca a ausência pelo defeito oposto.

---

## Parte 3 — Limites

Escritos como proibição porque são proibição.

| Limite | Regra |
|---|---|
| `gtm versions publish` | **Nunca**, em nenhuma circunstância. |
| `delete` de qualquer recurso | Só com confirmação explícita no chat, item a item. Padrão: pausar ou corrigir. |
| `--force` | Nunca. Existe para pular a confirmação, que é justamente o ponto. |
| Escrita fora do workspace de auditoria | Proibida, em especial no `Default`. |
| Correção sem achado | Proibida. Toda escrita cita o id do check que a justifica. |
| Achado de dev ou cliente | Sai como proposta. O container não é lugar de consertar o site. |
| Credenciais | A skill nunca digita senha nem edita `credentials.json`. Token expirado é parada. |

**Se algo falhar duas vezes, pare.** Reporte o que foi aplicado, o que não foi, e devolva ao humano
com o estado do workspace. Um workspace meio corrigido, documentado, é recuperável; um workspace
meio corrigido em silêncio, não.
