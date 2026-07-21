# Auditoria de Tracking — skills de integridade da coleta

Duas skills complementares para auditar a coleta de dados de contas de mídia paga (Google Ads,
Meta Ads, GA4). **Uma mede o sintoma, a outra acha a causa e corrige.**

## O eixo: camadas de evidência

A divisão entre as skills não é por ferramenta — é por **onde mora a evidência**. Cada camada
responde uma pergunta diferente, e nenhuma responde a da outra:

| Camada | Pergunta | Evidência | Onde vive |
|---|---|---|---|
| **Números** | o sintoma existe? | relatórios das plataformas e do GA4 | `integridade-coleta` |
| **Configuração** | o container declara certo? | a versão publicada do GTM | `diagnostico-gtm`, fase 2 |
| **Comportamento** | o site dispara certo? | as requisições reais do navegador | `diagnostico-gtm`, fase 3 |
| **Correção** | como fica certo? | o diff do workspace | `diagnostico-gtm`, fase 5 |

As três últimas moram juntas porque se completam: **config não é disparo, disparo não é config, e
corrigir sem ter provado a causa é chute.** Só a camada de números é separável — roda em minutos, em
qualquer conta, sem acesso ao GTM.

## Estrutura

```
skill-tracking-audit/
├── integridade-coleta/          # o termômetro — mede o sintoma nos números
│   ├── SKILL.md
│   └── references/
│       ├── limiares.md          #   calibragem editável (limiares, faixas, janelas)
│       └── queries.md           #   mecânica de leitura (GAQL, Meta, GA4)
├── diagnostico-gtm/             # o exame e a cirurgia — audita e corrige a instrumentação
│   ├── SKILL.md
│   └── references/
│       ├── gtm-cli.md           #   instalação, autenticação e comandos do gtm-cli
│       ├── checks.md            #   catálogo C · R · U, com o comando de cada check
│       └── correcoes.md         #   protocolo de escrita e receitas de correção
└── archive/                     # skills aposentadas — ver archive/README.md
    └── causa-raiz-tracking/
```

## As skills

### 1. [`integridade-coleta`](integridade-coleta/SKILL.md) — o termômetro

Sanity check de onboarding. Responde **"dá para confiar nos dados dessa conta?"** em ~10 minutos, só
com credencial de API (ou nem isso, no modo geral, que roda sobre exports).

- 13 checks com pré-condição: conversão ativa e viva, valor plausível, GCLID, parametrização anúncio
  a anúncio, discrepância contra o GA4
- Saída: veredito **APTA / NÃO APTA** + score ponderado por gasto exposto + recomendações em ordem
  de dinheiro comprometido
- Calibragem em [`limiares.md`](integridade-coleta/references/limiares.md), mecânica de leitura em
  [`queries.md`](integridade-coleta/references/queries.md)

### 2. [`diagnostico-gtm`](diagnostico-gtm/SKILL.md) — o exame e a cirurgia

Investigação sob demanda, sobre o [gtm-cli](https://github.com/owntag/gtm-cli). Responde **"por que
esse sintoma existe — e como fica certo?"**

Cinco fases: pré-voo (instala, autentica e configura o CLI) → descoberta (prova que o container
auditado é o do site) → configuração (lê a versão publicada e registra o que ela **prevê** que vai
disparar) → comportamento (navega o funil e confronta a predição) → UTMs por anúncio → correção.

- Pega o que nenhum modo isolado pega: o **disparo órfão** — o que a rede mostra e o container não
  explica (pixel hardcoded no tema, plugin), causa raiz clássica do duplo disparo
- Aplica as correções em **workspace dedicado** e cria a versão. **Nunca publica** — o humano testa
  em Preview e publica
- Cada achado nomeia o responsável (`analista` / `desenvolvedor` / `cliente`), e o dono decide o que
  a skill pode aplicar sozinha
- Saída: nível de integridade em escala de 1 a 5, fila de correções, diff literal do workspace, e o
  roteiro de teste

## O ciclo em produção

```
integridade-coleta  →  achou sintoma?  →  diagnostico-gtm  →  humano testa em Preview e publica
      (mede)                              (prova e corrige)              ↓
      ↑                                                                 │
      └──────────────  confirma que resolveu, 7 dias depois  ────────────┘
```

Score limpo na primeira? A segunda nem entra em campo. São separadas de propósito: a primeira roda
em qualquer conta hoje, só com API; a segunda depende de acesso ao GTM e a um navegador — fundi-las
faria a rápida herdar a fragilidade da lenta.

## Princípios comuns

- **Nunca `PASSOU` no que não foi verificado.** `NAO_VERIFICADO` e `NAO_APLICAVEL` são respostas
  honestas; aprovação falsa não.
- **Sintoma não é diagnóstico.** "Compatível com duplo disparo — verifique" na primeira; o flagrante
  com URLs de requisição na segunda.
- **Nunca corrija o que não provou.** Toda escrita no container cita o id do check que a justifica.
- **Aplica, nunca publica.** A correção fica pronta em workspace isolado; quem põe no ar é o humano,
  depois de testar.
- **Pausar, não apagar.** Reversível em um comando, e preserva a configuração para quem investigar
  depois.
- **Fonte faltando reduz cobertura, não aborta a skill** — e a cobertura vai declarada no relatório.
- **Recomendação primeiro, explicação recolhida.** O porquê existe para quem quiser abrir.
- Output com bloco estruturado (id, severidade, gasto exposto, ação, responsável), pronto para virar
  cartão de aceite numa fila de produção.

## Status

A `integridade-coleta` foi escrita contra a documentação das APIs (GAQL, Meta Marketing API, GA4
Data API) e **ainda não foi validada contra conta real**. Primeiro ponto a testar: os nomes de campo
do recurso `conversion_action` no Google Ads.

A `diagnostico-gtm` v2.0 teve seus comandos verificados contra o **gtm-cli v1.5.8**; falta a
validação ponta a ponta da fase de correção em container descartável. Até lá, revise o diff do
workspace antes de publicar qualquer versão que ela criar.

As skills instruem reportar `NAO_VERIFICADO` em vez de improvisar quando um campo ou comando não
existir.
