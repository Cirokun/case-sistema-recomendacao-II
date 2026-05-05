# Report Executivo — Sistema de Recomendação de Produtos Financeiros

## Abordagem Escolhida

**LambdaMART via LightGBM** — algoritmo de Learning to Rank que otimiza diretamente métricas de ordenação (NDCG). A escolha se justifica por três razões: (1) o problema é fundamentalmente de *ranking* — apenas as 5 primeiras posições do carrossel geram resultado, então otimizar classificação binária seria subótimo; (2) LightGBM é eficiente em dados tabulares com 500k registros e 35+ features; (3) features ricas de perfil do cliente (renda, segmento, histórico, investimentos) não seriam aproveitadas por filtragem colaborativa pura.

O modelo foi treinado com **split temporal**: safras 2024-01 a 2025-09 como treino (~87.5%) e as 3 últimas safras (2025-10 a 2025-12) como teste, evitando vazamento de dados temporais.

## Principais Descobertas do EDA

- **Bias de posição é forte**: CTR nas posições 1–5 é ~2.8× maior que nas posições 6–20, confirmando a importância de rankear bem os primeiros itens.
- **Segmentação importa**: clientes premium têm 3× mais investimentos e 2× mais renda que básicos; os produtos mais contratados variam significativamente por segmento.
- **`app_busca` converte melhor** que outros canais de interação — cliente com intenção ativa tem propensão 40% maior.
- **Sazonalidade estável**: volume de interações cresce ~8% a.a., sem picos bruscos intraano nos dados sintéticos.
- **Co-contratação**: `credito_pessoal` + `cheque_especial` é o par mais frequente; `consorcio_imovel` raramente coexiste com `titulo_capitalizacao`.

## Resultados Alcançados

| Modelo | Precision@5 | NDCG@5 | Hit Rate@5 |
|---|---|---|---|
| Aleatório | ~0.05 | ~0.08 | ~0.23 |
| Popularidade (baseline) | ~0.08 | ~0.12 | ~0.35 |
| **LambdaMART** | **~0.14** | **~0.21** | **~0.55** |

O modelo LambdaMART supera o baseline de popularidade em **~75% no NDCG@5** e **~57% no Hit Rate@5** — indicando que a personalização por cliente gera ganho substancial versus uma lista estática por popularidade.

## Limitações e Próximos Passos

**Limitações:** cold start para clientes sem histórico; position bias parcialmente corrigido como feature mas não pela função objetivo; dados sintéticos podem não capturar todos os padrões reais.

**Próximos passos:** (1) Two-Tower Neural Network para representação mais rica; (2) Contextual Bandit para exploração de produtos com poucos dados; (3) feature store em tempo real para capturar comportamento intraday; (4) A/B test em produção com acompanhamento de receita incremental por posição.
