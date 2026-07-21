---
name: diagnostico-gtm
version: 2.0
description: Auditoria e correção do Google Tag Manager como causa raiz da coleta, via gtm-cli. A camada de números (integridade-coleta) mede o sintoma — discrepância, sessões não atribuídas, valor ausente; esta skill audita a causa na instrumentação e aplica a correção no container. Cinco fases — pré-voo (instala e autentica o gtm-cli), descoberta, configuração (lê a versão publicada pela API), comportamento (navegação real capturando os disparos de GA4, Google Ads e pixel Meta até o checkout) e UTMs por anúncio — fechando em correções aplicadas em workspace isolado, com a versão criada e pronta para revisão. Aplica, nunca publica. Use quando pedirem para auditar o GTM, verificar tags e dataLayer, entender por que uma tag não dispara, investigar duplo disparo ou sessões caindo em direct, ou corrigir o container.
author: Arthur.OS (Henrique)
output_format: Arquivo HTML autocontido no padrão visual Arthur.OS + bloco JSON estruturado (causas × sintomas × correções × execução), gravados na pasta da conta
---

# Diagnóstico · GTM — causa raiz da coleta, e a correção

A camada de números responde "dá para confiar nos números?". Esta skill responde as duas perguntas
que vêm quando a resposta é não: **onde, na instrumentação, a coleta quebra — e como fica certo?**

## Lugar no conjunto

Irmã de [`integridade-coleta`](../integridade-coleta/SKILL.md), com divisão por **camada de
evidência**:

| Camada | Pergunta | Onde vive |
|---|---|---|
| Números | o sintoma existe? | `integridade-coleta` |
| Configuração | o container declara certo? | aqui, fase 2 |
| Comportamento | o site dispara certo? | aqui, fase 3 |
| Correção | como fica certo? | aqui, fase 5 |

As três últimas são uma skill só porque se completam: config não é disparo, disparo não é config, e
corrigir sem ter provado a causa é chute. Só a camada de números é separável — roda em minutos, sem
acesso ao GTM.

Roda tipicamente **depois** da camada de números, alimentada pelos achados dela: cada achado
`fora_de_escopo` com hipótese em tag ou site vira uma hipótese a confirmar ou descartar. Também roda
avulsa ("audita o GTM dessa conta") — as varreduras são as mesmas, só não há sintomas para cruzar.

## Aplica, nunca publica

A skill escreve no container: cria e corrige tags, gatilhos e variáveis — **em workspace dedicado**,
nunca no `Default`, e fecha criando a versão. **`gtm versions publish` está fora de escopo, sem
exceção.** Quem publica é o humano com acesso, depois de testar em Preview.

O que isso significa na prática: nada do que a skill faz afeta o site até alguém publicar. Apagar o
workspace de auditoria devolve o container ao estado inicial.

## O que esta skill NÃO é

- **Não publica.** Nem `versions publish`, nem `--force`, nem `delete` sem confirmação item a item.
  O padrão para tag defeituosa é pausar ou corrigir.
- **Não conserta o site.** Achado cuja correção mora no código, no tema ou no dataLayer sai como
  **proposta com dono nomeado**, não como escrita no container.
- **Não substitui a camada de números.** Os números (discrepância, score, portão) continuam lá;
  aqui se explica o que eles apontaram.
- **Não valida purchase em runtime.** Comprar de verdade não está no roteiro. O purchase se resolve
  pela escada de [`checks.md`](references/checks.md) — pedido de teste combinado, staging ou
  triangulação declarada. Afirmar "purchase testado no site" sem transação é mentira metodológica.
- **Não audita consentimento juridicamente.** Lê o consent mode como mecânica de disparo (o que
  bloqueia o quê), não como conformidade LGPD.
- **Não manipula credenciais.** Token expirado é parada com pedido explícito ao humano.

---

## As cinco fases

Referências: [`gtm-cli.md`](references/gtm-cli.md) (instalação, auth, comandos),
[`checks.md`](references/checks.md) (catálogo C · R · U),
[`correcoes.md`](references/correcoes.md) (protocolo de escrita e receitas).

### Fase 0 — Pré-voo

```bash
gtm --version          # instalado? → siga
gtm auth status        # autenticado? → siga
```

Se ausente: script de instalação → npm → binário da release. Se não autenticado: `gtm auth login`
(OAuth) ou `--service-account`. Depois, sempre:

```bash
gtm config set outputFormat json
gtm config set defaultAccountId <id>
gtm config set defaultContainerId <id>
gtm config set defaultWorkspaceId <id>
```

Sem autenticação, as fases 2 e 5 não rodam. A fase 3 (navegador) não depende delas — siga com a
cobertura declarada. **Fonte faltando reduz cobertura, não aborta a skill.**

### Fase 1 — Descoberta

`gtm accounts list` → `gtm containers list`, casando o `publicId` (`GTM-XXXXXX`) com o que aparece
no fonte do site. É o que prova que o container auditado é o que a página carrega.

### Fase 2 — Configuração (C01–C10)

`gtm versions live -o json` — a versão **no ar**, numa chamada. Dez checks: cobertura de
plataformas, fiação do purchase, `transaction_id`, Conversion Linker, duplicidade interna, tags
pausadas, consent como mecânica, gatilhos frágeis, linha do tempo de versões, e o handoff do
dataLayer à fase 3.

**Audite sempre a `versions live`, nunca o workspace** — workspace é intenção, live é o que o site
roda. O rascunho entra como medida de drift (`gtm workspaces status`): quantas mudanças pendentes,
e quais.

**A fase fecha registrando as predições** — o que a config prevê que vai disparar em cada passo do
roteiro. Obrigatório: é o antídoto do viés de confirmação que a ordem config→runtime introduz, e é
o que transforma a fase 3 em teste em vez de passeio.

### Fase 3 — Comportamento (R01–R07)

Navegação instrumentada — browser headless ou Claude in Chrome — no roteiro
**home → produto → adicionar ao carrinho → carrinho → início do checkout**, capturando as chamadas
de rede (`/g/collect`, `googleadservices`, `facebook.com/tr`), o dataLayer e o consentimento.

Sete checks: GA4 único por pageview, escada de e-commerce populada, pixel Meta único e com dedup,
cookies de atribuição, sobrevivência da UTM, consent na prática, latência de carregamento.

Cada observação é confrontada com a predição da fase 2. **Divergência tem nome:** predito e não
observado = config que não dispara; observado e não predito = **disparo órfão** (hardcode, plugin).
O disparo órfão é a causa raiz clássica do duplo disparo — o container parece impecável e a conta
conta tudo em dobro.

### Fase 4 — UTMs por anúncio (U)

Varredura dos parâmetros de URL de todos os anúncios ativos com gasto, em três estados de
severidade deliberadamente distinta: **correta** (verde), **fora das melhores práticas** (amarelo —
o número existe, só está desorganizado; recuperável por padrão) e **inexistente** (vermelho — o GA4
fica cego ao canal e nada recupera retroativamente). Sempre em dois níveis: agregado e lista
nominal. Exceção do `gclid` no Google.

A correção é no gerenciador de anúncios, não no container: sai sempre como proposta.

### Fase 5 — Correção

Sete passos obrigatórios, detalhados em [`correcoes.md`](references/correcoes.md):

1. **Backup** de live, tags, triggers e variables na pasta da conta.
2. **Workspace dedicado** — `gtm workspaces create --name "auditoria-AAAA-MM-DD"`.
3. **`gtm workspaces sync`** — conflito com a live é parada.
4. **Inspecionar antes de criar** — herdar o formato de uma tag que já funciona.
5. **Aplicar**, uma correção por vez, cada uma amarrada ao achado que a justifica.
6. **Diff** — `gtm workspaces status`, transcrito como lista literal no relatório.
7. **`gtm versions create`** com notas — e **PARA**.

**Só se aplica o que tem `responsavel: analista`** — o que se resolve dentro do container. Achado de
`desenvolvedor` (dataLayer, código) ou de `cliente` (acesso, decisão) sai como proposta.

---

## Cruzamento — sintoma × causa × correção

O produto central quando há auditoria de números anterior: cada sintoma encaminhado recebe um
veredito de causa.

- Cada achado carrega `explica: [...]` — os ids dos sintomas que ele explica.
- Sintoma que **nenhum** achado explica fica dito com todas as letras: "causa não encontrada no GTM
  — investigar plataforma/servidor" é resultado honesto e útil, não fracasso.
- Hipótese **descartada** também é entrega ("duplo disparo descartado: pixel único confirmado em
  runtime") — fecha porta, poupa investigação.
- Cada achado nomeia o **responsável**: `analista` (resolve na plataforma ou no container),
  `desenvolvedor` (código, dataLayer) ou `cliente` (acesso, decisão). Achado sem dono é achado que
  ninguém executa — e o dono decide o que a skill pode aplicar sozinha.
- Cada correção carrega `status_execucao`: `aplicada` (no workspace), `proposta` (fora do alcance do
  container) ou `bloqueada` (exigiria delete ou publish — aguarda decisão humana).

## Output

Duas entregas, na pirâmide de atenção do conjunto (camada 1 decide · camada 2 explica · camada 3
prova):

1. **Bloco JSON** — contrato: fases rodadas, cobertura por check, achados com `explica` e
   `responsavel`, correções com `status_execucao`, hipóteses descartadas, e os objetos
   `operabilidade` e `execucao`.
2. **HTML autocontido no padrão visual Arthur.OS** (azul elétrico `#0F0EFE`, navy `#0B0078`,
   Space Grotesk/Inter/JetBrains Mono, chips por sinal), gravado na pasta da conta
   (`diagnostico/AAAA-MM-DD-gtm.html`).

**O topo da camada 1 é uma pergunta objetiva com resposta em escala, mais os passos — nada além:**

1. **A pergunta e a nota.** "Qual o nível de integridade da coleta?" — respondida na **escala de
   1 a 5**, com o rótulo e uma frase do que isso significa na prática:

   | Nível | Rótulo | O que permite |
   |---|---|---|
   | 1 | **Integridade nula** | Nenhum número serve; não decida nada por dados. |
   | 2 | **Integridade baixa** | Defeitos graves; dá para inferir tendências, não para decidir. |
   | 3 | **Integridade suficiente** | Existe uma referência confiável; dá para operar por ela, com ressalvas nas demais fontes. |
   | 4 | **Integridade boa** | Os números principais estão certos em todas as plataformas; restam refinamentos. |
   | 5 | **Integridade total** | Tudo confiável; nada a corrigir. |

   A nota sai com escala visual (5 segmentos preenchidos até o nível) e a frase prática do nível —
   ex.: "**3 de 5 · Integridade suficiente** — as vendas do GA4 estão certas; use-as como
   referência. Meta e Google, por enquanto, são tendência."
2. **O que falta para subir de nível**, em passos contáveis: "Para chegar ao nível 4, são 3
   passos: …". Sempre ancorado no próximo nível, não no 5 — o caminho inteiro mora na fila e nas
   camadas de baixo.
3. **O que já foi corrigido e o que falta para valer.** Quando a fase 5 rodou, a camada 1 diz, em
   linguagem simples, que as correções estão prontas mas **ainda não estão no ar**, e qual é o
   próximo passo humano: testar e publicar.

Definição interna de "certo" (guia o julgamento, **não é verbalizada no relatório**): o estado que
entrega os números de que a operação precisa — canal legível no GA4, uma compra contada uma vez com
valor certo, um pixel por plataforma, conversão do Ads com valor e dedup. Melhorias além disso
(sGTM, CAPI, consent avançado) não entram no topo nem atrasam o veredito; se citadas, vão para a
camada 3.

**Regra de linguagem da camada 1: didática e palpável.** Sem jargão, sem sigla de check, sem nome de
tag, sem termo de plataforma que um gestor não-técnico não reconheça. "O site conta cada venda duas
vezes para o Meta" — não "Purchase duplicado sem eventID". O vocabulário técnico inteiro (ids, tags,
checks, parâmetros, comandos) vive nas camadas 2 e 3, onde quem executa precisa dele. Cada frase da
camada 1 deve passar no teste: o dono da loja entende sem perguntar nada?

Depois do topo: a fila de correções (máx. 5, uma linha imperativa cada, com o que cada uma resolve
em linguagem simples), marcando as que já estão aplicadas no workspace. Camada 2: cada correção
expande em problema → mudança → efeito esperado → como testar, e a varredura U com sua lista
nominal. Camada 3: checklist C/R/U completo com evidência (request capturado, trecho do container,
url_tags), o diff literal do workspace, hipóteses descartadas, não verificados com o que faltou,
carimbo.

No JSON, os dois objetos de contrato:

```json
"operabilidade": {
  "nivel_integridade": 3,
  "rotulo": "suficiente",
  "regua_atual": "ga4_transactions",
  "ressalvas": ["nativos como tendência, não veredito"],
  "passos_para_proximo_nivel": ["publicar a versão criada", "corrigir padrão de UTM no Ads Manager", "rebaixar ação de conversão duplicada"],
  "alem_do_proximo_nivel": ["reativar CAPI via sGTM"]
},
"execucao": {
  "workspace_id": "12",
  "workspace_nome": "auditoria-2026-07-21",
  "version_id": "43",
  "publicado": false,
  "backup": "diagnostico/2026-07-21-backup-live.json",
  "correcoes_aplicadas": 4,
  "correcoes_propostas": 2,
  "correcoes_bloqueadas": 1,
  "instrucoes_publicacao": "GTM → Preview no workspace auditoria-2026-07-21 → roteiro home→checkout → publicar em janela de baixo tráfego"
}
```

## Guias

- **Nunca afirme que uma tag dispara porque ela existe no container** — nem que não existe porque
  não disparou no seu roteiro. Config prova intenção; runtime prova comportamento; a afirmação usa
  a evidência do modo que a sustenta.
- **Predição antes de navegar.** A fase 3 sem as predições da fase 2 escritas vira busca por
  confirmação do que já se acredita.
- **Evidência carrega o artefato.** Achado de runtime cita a request capturada (URL, evento,
  parâmetros); achado de config cita a tag pelo nome e id no container; correção aplicada cita o
  comando e o diff. "O pixel duplica" sem o request é opinião.
- **Nunca `PASSOU` no que não foi verificado.** `NAO_VERIFICADO` com o motivo é resposta honesta;
  aprovação falsa não é.
- **Nunca corrija o que não provou.** Toda escrita no container cita o id do check que a justifica.
  Correção sem achado é mudança não solicitada num sistema de produção.
- **Pausar, não apagar.** Pausar é reversível em um comando e preserva a configuração para quem
  investigar depois.
- **Sites variam; o roteiro se adapta.** Single-page app, checkout em domínio próprio, headless
  commerce — o roteiro segue o funil real da loja, e passos impossíveis viram `NAO_VERIFICADO` com
  motivo, nunca silêncio.
- **Duas tentativas e para.** Se o mesmo comando ou passo de navegação falhar duas vezes, reporte o
  que foi coletado e aplicado, com a cobertura honesta, e devolva ao humano. Um workspace meio
  corrigido e documentado é recuperável; em silêncio, não.

## Versão e histórico

**v1.0** — Nasce como bloco irmão do 1 no conjunto de diagnóstico, movendo a fronteira que a coleta
mantinha ("não inspeciona GTM") para uma skill própria. Dois modos complementares: config (export
JSON do container, com preferência à versão publicada, ou API readonly; checks C01–C10) e runtime
(navegação instrumentada home→checkout; checks R01–R07). Purchase por triangulação, nunca "testado".
Produto central: cruzamento sintoma × causa × correção com o contrato do Bloco 1 (`explica`),
incluindo hipóteses descartadas como entrega. Output na pirâmide de atenção, JSON + HTML Arthur.OS.
Motivada em campo pela rodada Space Adventure Canela (07/2026): connect 57%, X01 a +36% do limite e
estouro do X02 — três sintomas cuja causa mora exatamente onde a coleta não enxergava.

**v1.1** — Calibrada na rodada Zapalla (07/2026). Dois acréscimos. (1) **Modo U — UTMs por
anúncio**, em três estados de severidade deliberadamente distinta, sempre em dois níveis (agregado
+ lista nominal), com a exceção do gclid no Google. (2) **Topo de operabilidade**: a camada 1 abre
respondendo se dá para otimizar hoje e aponta o cenário ótimo como suficiência, não maturidade
máxima. JSON ganha o objeto `operabilidade`.

**v1.2** — Ajuste de forma no topo, por feedback de campo. O topo vira **duas afirmações curtas**:
se e como operar hoje, e o que falta em passos contáveis. A definição de "ótimo como suficiência"
vira guia interno e sai do texto do relatório. Nasce a **regra de linguagem da camada 1**: didática
e palpável, sem jargão, sem sigla, sem nome de tag; o teste é o dono da loja entender sem perguntar.

**v1.3** — O topo ganha forma final: **pergunta objetiva + resposta em escala de 1 a 5**, com
rótulo simétrico (nula · baixa · suficiente · boa · total) e a frase prática do que o nível permite;
em seguida, os passos contáveis para subir ao **próximo** nível. `operabilidade` passa a carregar
`nivel_integridade`, `rotulo` e `passos_para_proximo_nivel`.

**v2.0** — A skill deixa de só propor e passa a **executar**, sobre o
[gtm-cli](https://github.com/owntag/gtm-cli) (owntag GmbH, MIT). Quatro mudanças estruturais.

(1) **Reorganização da família por camada de evidência**, não por ferramenta. Separar skills por
ferramenta (export / API / navegador) fazia com que se canibalizassem: a `causa-raiz-tracking` já
estava inteiramente contida nesta, e o modo C por export estava contido no que o CLI faz melhor. O
conjunto passa a ter duas skills — números (`integridade-coleta`) e instrumentação (esta) — e a
`causa-raiz-tracking` foi arquivada depois de doar o campo `responsavel` e o protocolo de purchase
por pedido de teste.

(2) **Os modos viram cinco fases ordenadas**, com pré-voo explícito (instalação, autenticação e
configuração do CLI) e descoberta que **prova** que o container auditado é o do site, casando o
`publicId` com o fonte da página. O modo C vira CLI-first: `gtm versions live` elimina a ambiguidade
"isso é a versão publicada ou o rascunho?" que o export manual carregava como ressalva, e
`gtm workspaces status` transforma o drift do rascunho em achado medido. Export manual continua
aceito, como degradação declarada, quando não há acesso OAuth.

(3) **Fase 3 dirigida por predição.** A ordem config→runtime é mais barata e mais rápida, mas
enviesa: quem viu o container procura confirmá-lo. A fase 2 passa a fechar registrando o que prevê
que vai disparar, e a fase 3 confronta — nomeando as divergências nas duas direções, inclusive o
disparo órfão que nenhum dos modos isolados pegava.

(4) **Fase 5 — correção aplicada.** Protocolo de sete passos com backup, workspace dedicado, sync,
inspeção antes de criar, diff literal e versão criada com notas. A fronteira nova é
`gtm versions publish`, **nunca** — a skill entrega a versão pronta e o roteiro de teste em Preview;
publicar segue humano. Achado de `desenvolvedor` ou `cliente` continua saindo como proposta: o
container não é lugar de consertar o site. JSON ganha o objeto `execucao` e o campo
`status_execucao` por correção.
