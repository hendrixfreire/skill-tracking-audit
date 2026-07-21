# Arquivo

Skills aposentadas. Ficam aqui pelo registro — **não devem ser usadas**.

## `causa-raiz-tracking` — aposentada em 21/07/2026

Sucedida por [`diagnostico-gtm`](../diagnostico-gtm/SKILL.md) v2.0.

**Por quê.** A `diagnostico-gtm` já a continha inteira. Os checks de container da causa-raiz
(`C01`–`C05`) são um subconjunto dos `C01`–`C10` da sucessora; os de rede (`N01`–`N05`) são um
subconjunto dos `R01`–`R07`; o cruzamento (`R01`/`R02`) virou a confrontação predição × observação
da fase 3. A sucessora ainda acrescenta a varredura de UTMs, o topo de operabilidade em escala e a
fase de correção aplicada.

**A causa da redundância** — vale registrar, porque é a lição que reorganizou o conjunto: as skills
estavam separadas por **ferramenta** (API do GTM / navegador / export), e ferramentas se sobrepõem.
O eixo que não se sobrepõe é a **camada de evidência**: números, configuração, comportamento,
correção. Duas skills bastam.

**O que foi migrado antes de arquivar:**

- **O campo `responsavel`** em cada achado — `analista`, `desenvolvedor` ou `cliente`. Achado sem
  dono é achado que ninguém executa. Na sucessora ganhou função nova: o dono decide o que a skill
  pode aplicar sozinha no container e o que sai como proposta.
- **O protocolo de purchase por pedido de teste** — pedido real combinado com o cliente (cupom de
  100%), com horário avisado, observando a rede durante o checkout. Virou a primeira opção da escada
  em [`checks.md`](../diagnostico-gtm/references/checks.md); staging é a segunda; a triangulação, a
  terceira.
- **O mapeamento de tipos do GTM** (`gaawc`, `gaawe`, `awct`, `sp`, `html`, `img`; variáveis
  `c`/`v`/`jsm`/`u`/`k`) de `references/gtm-api.md`, hoje em
  [`gtm-cli.md`](../diagnostico-gtm/references/gtm-cli.md). O mapeamento não mudou — mudou a
  ferramenta que faz a leitura.

O `references/navegacao.md` (protocolo de navegação e leitura de rede) continua válido como
material de apoio à fase 3 da sucessora.
