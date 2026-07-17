# Auditoria de Tracking — skills de integridade da coleta

Duas skills complementares para auditar a coleta de dados de contas de mídia paga
(Google Ads, Meta Ads, GA4). **Uma mede, a outra prova.**

## As skills

### 1. [`integridade-coleta`](SKILL.md) — o termômetro

Sanity check de onboarding. Responde **"dá para confiar nos dados dessa conta?"** em ~10 minutos,
só com credencial de API (ou nem isso, no modo geral, que roda sobre exports).

- 13 checks com pré-condição: conversão ativa e viva, valor plausível, GCLID, parametrização
  anúncio a anúncio, discrepância contra o GA4
- Saída: veredito **APTA / NÃO APTA** + score ponderado por gasto exposto + recomendações em
  ordem de dinheiro comprometido
- Calibragem em [`references/limiares.md`](references/limiares.md), mecânica de leitura em
  [`references/queries.md`](references/queries.md)

### 2. [`causa-raiz-tracking`](gtm-deep-audit/SKILL.md) — o exame

Investigação sob demanda. Responde **"por que esse sintoma existe?"** quando a primeira skill
aponta algo suspeito.

- Lê o container publicado do GTM via Tag Manager API e observa as tags disparando na rede do
  navegador (extensão do Chrome): duplo disparo em flagrante, pixel hardcoded, valor errado no
  dataLayer, CAPI sem `event_id`
- Saída: cada hipótese vira **CONFIRMADO / REFUTADO / INCONCLUSIVO**, com evidência conferível
  e responsável nomeado (analista / desenvolvedor / cliente)
- API do GTM em [`gtm-deep-audit/references/gtm-api.md`](gtm-deep-audit/references/gtm-api.md),
  protocolo de navegação em
  [`gtm-deep-audit/references/navegacao.md`](gtm-deep-audit/references/navegacao.md)

## O ciclo em produção

```
integridade-coleta  →  achou sintoma?  →  causa-raiz-tracking  →  correções com dono  →  integridade-coleta de novo
      (mede)                                   (prova)                                    (confirma que resolveu)
```

Score limpo na primeira? A segunda nem entra em campo. São separadas de propósito: a primeira
roda em qualquer conta hoje, só com API; a segunda depende de acesso ao GTM e navegador —
fundi-las faria a rápida herdar a fragilidade da lenta.

## Princípios comuns

- **Só diagnóstico.** Nenhuma das duas executa correção; o output é recomendação com dono.
- **Nunca `PASSOU` no que não foi verificado.** `NAO_VERIFICADO` e `NAO_APLICAVEL` são respostas
  honestas; aprovação falsa não.
- **Sintoma não é diagnóstico.** "Compatível com duplo disparo — verifique" na skill 1; o
  flagrante com URLs de requisição na skill 2.
- **Recomendação primeiro, explicação recolhida.** O porquê existe para quem quiser abrir.
- Output com bloco estruturado (id, severidade, gasto exposto, ação, responsável), pronto para
  virar cartão de aceite numa fila de produção.

## Status

Escritas contra a documentação das APIs (GAQL, Meta Marketing API, GA4 Data API, Tag Manager
API v2), **ainda não validadas contra conta real**. Primeiro ponto a testar: os nomes de campo
do recurso `conversion_action` no Google Ads. As skills instruem reportar `NAO_VERIFICADO` em
vez de improvisar quando um campo não existir.
