# gtm-cli — instalação, autenticação e comandos

CLI oficial do Google Tag Manager mantida pela owntag GmbH (MIT).
Repo: <https://github.com/owntag/gtm-cli> · verificado contra a **v1.5.8**.

Substitui o export manual do container e a chamada REST crua: uma ferramenta lê a versão publicada,
mede o rascunho e escreve as correções. Traz `gtm agent guide` — guia embutido escrito para LLMs,
com os formatos de config e os erros conhecidos. **Em dúvida sobre um formato, rode
`gtm agent guide` antes de improvisar.**

**Regra que vale em todo este arquivo:** se um comando ou campo não existir como descrito aqui,
o check é `NAO_VERIFICADO` com o motivo. Nunca invente flag.

---

## 1. Instalação

Primeiro, verifique — na maioria das máquinas já está instalado:

```bash
gtm --version            # instalado? → siga para a autenticação
gtm upgrade --check      # só checa se há versão nova
gtm upgrade              # atualiza (--force reinstala mesmo já atual)
```

Se ausente, nesta ordem de preferência:

| Método | Comando |
|---|---|
| Script (macOS, Linux, WSL) | `curl -fsSL https://raw.githubusercontent.com/owntag/gtm-cli/main/install.sh \| bash` |
| npm | `npm install -g @owntag/gtm-cli` |
| Binário da release | `curl -fsSL https://github.com/owntag/gtm-cli/releases/latest/download/gtm-darwin-arm64 -o gtm && chmod +x gtm && sudo mv gtm /usr/local/bin/` |
| Deno, sem instalar | `deno run --allow-net --allow-read --allow-write --allow-env --allow-run https://raw.githubusercontent.com/owntag/gtm-cli/main/src/main.ts` |

O binário costuma cair em `~/.local/bin/gtm` — o executável chama-se `gtm`, não `gtm-cli`.
**Windows exige WSL**; não há binário nativo.

Build a partir do fonte exige credencial OAuth própria no Google Cloud Console
(`GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, redirect `http://localhost:8085/callback`, Tag Manager
API habilitada). Só vale a pena em CI dedicado — para auditoria, use um dos quatro métodos acima.

---

## 2. Autenticação

```bash
gtm auth status          # e-mail autenticado e validade do token
gtm auth login           # OAuth: abre o navegador, callback em localhost:8085
gtm auth logout          # revoga
```

Alternativa para automação sem navegador:

```bash
gtm auth login --service-account /caminho/key.json
# ou
export GOOGLE_APPLICATION_CREDENTIALS=/caminho/key.json
```

Credenciais ficam em `~/.config/gtm-cli/credentials.json`.

**A skill nunca digita senha, nunca lê e nunca edita o arquivo de credencial.** Token expirado ou
ausente é **parada com pedido explícito** ao humano para rodar `gtm auth login` — jamais uma
tentativa de contornar. Sem autenticação, toda a fase de configuração é `NAO_VERIFICADO`; a fase de
comportamento (navegador) não depende dela e pode seguir com a cobertura declarada.

A conta autenticada precisa de acesso ao container do cliente. Para auditar sem corrigir, permissão
de leitura basta; para a fase de correção, é preciso permissão de edição no container.

---

## 3. Configuração da sessão

Sempre, no início:

```bash
gtm config set outputFormat json      # obrigatório — todo parsing depende disso
gtm config set defaultAccountId   <id>
gtm config set defaultContainerId <id>
gtm config set defaultWorkspaceId <id>
gtm config get                        # confere
```

Fixar os defaults elimina `-a/-c/-w` de cada comando. Todos aceitam sobrescrita pontual:
`-a, --account-id` · `-c, --container-id` · `-w, --workspace-id` · `-o, --output {json|table|compact}`.

`gtm config setup` faz o mesmo de forma interativa — evite em sessão de agente, prefira os `set`.

---

## 4. Descoberta

```bash
gtm accounts list -o json
gtm containers list -o json
gtm containers get --container-id <id> -o json
gtm containers snippet --container-id <id>     # snippet de instalação
gtm workspaces list -o json
```

**Case o `publicId` (`GTM-XXXXXX`) com o que aparece no fonte do site.** É o que prova que o
container auditado é o que a página carrega — sem isso, a auditoria inteira pode estar apontada
para o container errado.

---

## 5. Leitura do container

### A fonte da verdade

```bash
gtm versions live -o json        # a versão NO AR — tag[], trigger[], variable[], builtInVariable[]
```

**Audite sempre a `versions live`, nunca o workspace.** Workspace é intenção; live é o que o site
roda. Uma chamada resolve a fase de configuração inteira.

### O rascunho, como medida de drift

```bash
gtm workspaces status -o json    # mudanças pendentes no workspace vs. a live
gtm workspaces preview -o json   # prévia das mudanças
gtm workspaces sync              # detecta conflito com a live
```

A diferença entre rascunho e publicado não é ruído: é um **achado medido** ("há 7 mudanças não
publicadas há 40 dias"), a coisa que a auditoria por export manual só conseguia carimbar como
ressalva.

### Histórico

```bash
gtm version-headers list -o json          # datas de publicação, leve
gtm version-headers list --include-deleted -o json
gtm versions get --version-id <id> -o json
```

Cruze as datas de publicação com os degraus de coleta detectados na camada de números: mudança
publicada às vésperas de uma queda é causa provável — **coincidência apontada, não certeza afirmada**.

### Recursos, um a um

Todos seguem o mesmo par `list` / `get`, com `-o json`:

| Comando | Para quê na auditoria |
|---|---|
| `gtm tags list` · `gtm tags get --tag-id N` | inventário e config de cada tag |
| `gtm triggers list` · `gtm triggers get --trigger-id N` | o que dispara cada tag |
| `gtm variables list` · `gtm variables get --variable-id N` | de onde vêm valor, currency, items |
| `gtm built-in-variables list` | built-ins habilitados |
| `gtm gtag-configs list` | configurações gtag |
| `gtm templates list` · `gtm templates get --template-id N` | templates customizados (pixel Meta mora aqui) |
| `gtm folders list` | organização |
| `gtm clients` · `gtm transformations` · `gtm zones` | server-side GTM, quando houver |
| `gtm environments list` | ambientes (preview, live) |
| `gtm user-permissions list` | quem tem acesso — útil para nomear responsável |

---

## 6. Mapa de tipos

O que a auditoria precisa reconhecer no campo `type` do JSON.

### Tags

| `type` | O que é |
|---|---|
| `googtag` | **Google tag** — a config atual do GA4 |
| `gaawc` | GA4 Configuration — formato **legado**, ainda comum em containers antigos |
| `gaawe` | GA4 Event |
| `awct` | Google Ads Conversion Tracking |
| `gclidw` | **Conversion Linker** — persiste o `gclid` em cookie |
| `sp` | Google Ads Remarketing — **não confunda com o Linker** |
| `html` | **HTML customizado** — onde pixels de terceiros vivem |
| `img` | Pixel de imagem |
| `pntr` | Pinterest |
| `cvt_{containerId}_{templateId}` | Tag de **template customizado** |

Confirme o censo de tipos antes de filtrar por qualquer um deles — containers reais misturam
gerações:

```bash
jq -r '[.tag[] | .type] | group_by(.) | map({(.[0]): length}) | add' /tmp/live.json
```

**O pixel Meta não tem tipo nativo.** Ele aparece como `html` com `fbq('init', ...)` /
`fbq('track', ...)` dentro do `parameter` de chave `html` — ou como `cvt_…` se o container usa o
template da comunidade. Para achar duplicidade, faça grep de `fbq(` no conteúdo HTML de todas as
tags: dois `fbq('init')` para o mesmo pixel ID é duplicação dentro do próprio container.

Campos que importam em cada tag: `name` (vai na evidência), `type`, `firingTriggerId[]`,
`blockingTriggerId[]`, `parameter[]` (onde vivem IDs de conversão, event name e valor), `paused`
(`true` = fora do disparo, mas **listar no inventário**), `fingerprint` (controle de concorrência).

### Gatilhos

⚠️ **A forma do tipo muda entre leitura e escrita.** A API devolve minúsculo camelCase; o CLI
espera MAIÚSCULO no `--type` do `create`. Filtrar leitura pela forma de escrita retorna vazio — e
vazio interpretado como "não há" é um `PASSOU` falso.

| Na leitura (JSON) | Na escrita (`--type`) | Uso |
|---|---|---|
| `pageview`, `domReady`, `windowLoaded` | `PAGEVIEW`, `DOM_READY`, `WINDOW_LOADED` | genérico — correto para tag base, **errado** para conversão |
| `customEvent` | `CUSTOM_EVENT` | específico — o esperado para conversão (evento do dataLayer) |
| `click`, `linkClick`, `formSubmission`, `elementVisibility` | `CLICK`, `FORM_SUBMISSION` | frágil para evento crítico: quebra em redesign, silenciosamente |

```bash
jq -r '[.trigger[].type] | group_by(.) | map({(.[0]): length}) | add' /tmp/live.json
```

Tag de conversão de compra pendurada em `PAGEVIEW` de All Pages dispara em toda página, não na
compra. É achado imediato.

### Variáveis

| `type` | O que é |
|---|---|
| `c` | Constante |
| `v` | **Data Layer Variable** |
| `jsm` | Custom JavaScript |
| `u` | URL |
| `k` | Cookie primário |

Siga o parâmetro de valor da tag de conversão (`conversionValue` no `awct`; o `value` dentro do
`fbq('track','Purchase', {...})` no `html`). Ele deve referenciar uma `{{variável}}` do tipo `v`.
**Valor literal fixo (`1`, `0`, vazio) é violação** — é a origem, dentro do container, do sintoma
"toda compra vale R$ 1".

---

## 7. Escrita

Só depois da auditoria, só em workspace dedicado, só com achado que justifique. O protocolo
completo está em [`correcoes.md`](correcoes.md) — aqui ficam apenas as assinaturas.

```bash
gtm workspaces create --name "auditoria-AAAA-MM-DD"
gtm workspaces sync                                   # conflito? pare
```

```bash
gtm tags create --name "<nome>" --type <type> \
  --firing-trigger-id <ids> [--blocking-trigger-id <ids>] [--paused] \
  --config "$(cat /tmp/config.json)"

gtm tags update --tag-id <id> [--name ...] [--paused true] \
  [--firing-trigger-id <ids>] [--fingerprint <fp>] \
  --config "$(cat /tmp/config.json)"        # o config é MESCLADO com o existente

gtm triggers  create --name "<nome>" --type <TYPE> --config '{...}'
gtm variables create --name "<nome>" --type <c|v|jsm|u|k> --config '{...}'
```

```bash
gtm workspaces status -o json                         # o diff, para o relatório
gtm versions create --name "auditoria AAAA-MM-DD" --notes "<as correções>"
```

### Proibições

| Comando | Regra |
|---|---|
| `gtm versions publish` | **Nunca.** A publicação é do humano, depois do teste em Preview. |
| `gtm tags delete`, `triggers delete`, `variables delete`, `templates delete` | Só com confirmação explícita no chat, item a item. O padrão para tag defeituosa é **pausar ou corrigir**. |
| `--force` | Nunca. Existe para pular confirmação; a confirmação é justamente o ponto. |
| `gtm versions set-latest`, `undelete`, `delete` | Fora de escopo. |
| Qualquer escrita fora do workspace de auditoria | Proibida — em especial no `Default`. |

### Desfazer

```bash
gtm tags revert --tag-id <id>              # desfaz dentro do workspace
gtm triggers revert --trigger-id <id>
gtm variables revert --variable-id <id>
gtm built-in-variables revert
```

Apagar o workspace de auditoria devolve o container ao estado inicial — nada do que a skill fez
sobrevive fora dele até alguém publicar.

---

## 8. Padrões de execução

**JSON complexo vai em arquivo, nunca inline.** Escaping na linha de comando é a origem de metade
dos erros:

```bash
cat > /tmp/config.json << 'EOF'
{
  "parameter": [
    {"type": "TEMPLATE", "key": "name", "value": "ecommerce.value"}
  ]
}
EOF
gtm variables create --name "DLV - ecommerce.value" --type v --config "$(cat /tmp/config.json)"
```

**Inspecione antes de criar.** Antes de montar uma tag nova, leia uma equivalente que já funciona e
herde o formato exato de `type` e `parameter`:

```bash
gtm tags get --tag-id <id-de-uma-que-funciona> -o json
```

Vale sobretudo para templates customizados, onde o `type` só se descobre olhando.

**Paralelize o que é independente**, serialize o que depende de id alheio:

```bash
gtm variables create --name "V1" --type v --config "$(cat /tmp/v1.json)" &
gtm variables create --name "V2" --type v --config "$(cat /tmp/v2.json)" &
wait
```

**Extraia ids com `jq`**, nunca do output em tabela:

```bash
gtm triggers list -o json | jq -r '.[] | select(.name=="CE - purchase") | .triggerId'
```

**Nomenclatura** (a do guia oficial, seguida pela skill): `DLV - ecommerce.value` para data layer,
`CJS - <o que faz>` para JavaScript, `CE - purchase` para gatilho de evento customizado,
`<Plataforma> - <Evento>` para tag.

---

## 9. Troubleshooting

Os cinco erros documentados no `gtm agent guide`, em ordem de frequência:

| Erro | Causa | Correção |
|---|---|---|
| `Unknown entity type (template public ID: ...)` | tipo errado em tag de template customizado | use `cvt_{containerId}_{templateId}` — nem `cvt_26`, nem `26` |
| `Custom-event trigger must have exactly one custom-event filter` | `customEventFilter` ausente ou malformado | `arg0` **precisa** ser `{{_event}}`, e `arg1` o nome do evento |
| `The value must not be empty` | parâmetro com `"value": ""` | preencha ou remova o parâmetro |
| `Tag references an unknown trigger` | id de gatilho inexistente | `gtm triggers list` antes, e use o id real |
| `Bad escaped character in JSON` | escaping inline | heredoc em arquivo temporário |

Estrutura correta de um gatilho de evento customizado:

```json
{
  "customEventFilter": [{
    "type": "EQUALS",
    "parameter": [
      {"type": "TEMPLATE", "key": "arg0", "value": "{{_event}}"},
      {"type": "TEMPLATE", "key": "arg1", "value": "purchase"}
    ]
  }]
}
```

Diagnóstico quando algo falha, em ordem: `gtm auth status` → `gtm config get` →
`gtm tags get --tag-id <uma-que-funciona> -o json` (compare a estrutura) → teste com um recurso
trivial antes de repetir em lote.

**Duas tentativas e para.** Se o mesmo comando falhar duas vezes, reporte o que foi coletado com a
cobertura honesta e devolva ao humano. Insistir queima tempo e não muda o resultado.

---

## 10. Quando o CLI não está disponível

Sem acesso OAuth ao container, a fase de configuração roda em **modo degradado**: export manual do
container (GTM → Admin → Exportar contêiner), com os mesmos checks.

Ao exportar, é preciso escolher a **versão publicada (ao vivo)**, não o rascunho do workspace — e
como não há como confirmar o que foi escolhido, a auditoria carrega a ressalva carimbada. O CLI
existe justamente para eliminar essa ambiguidade. **Degradação declarada, nunca silenciosa:** o
relatório diz qual fase rodou por export e o que isso custou em confiabilidade.
