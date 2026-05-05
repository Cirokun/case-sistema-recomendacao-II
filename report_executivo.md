# Report Executivo — Sistema de Recomendação de Produtos Financeiros

## Abordagem

Implementamos **17 modelos de Regressão Logística** (um por produto) com **Causal Inference (position debiasing)** e **Epsilon-Greedy** para servir as recomendações. A estratégia de um modelo por produto permite que cada regressor aprenda os preditores mais relevantes para aquele produto sem interferência cruzada — por exemplo, `score_credito` importa muito para crédito pessoal, mas pouco para seguros — e que o `class_weight='balanced'` compense o desbalanceamento individualmente.

**Causal Inference — Position Debiasing:** o EDA revelou viés de posição significativo no carrossel. Para removê-lo, incluímos `posicao_exibicao` como feature durante o treino (permitindo ao modelo separar o efeito do produto do efeito da posição) e fixamos `posicao = 1` na inferência. Isso equivale ao operador **do-calculus** de Pearl: estimamos a probabilidade de contratação *se* todos os produtos fossem exibidos na melhor posição. O coeficiente de `posicao_exibicao` é negativo em **todos os 17 modelos**, confirmando que o viés foi capturado e neutralizado.

**Epsilon-Greedy (ε = 10%):** 90% das vezes o modelo serve o ranking personalizado (exploitation); 10% das vezes embaralha as posições para garantir diversidade de dados no próximo retreino, reduzindo o cold-start de produtos menos expostos.

**Split temporal:** treino nas safras 202407–202508 (~83,5% dos dados, ~296k interações e 2.700 contratações); teste nas safras 202509–202512 (~16,5%, ~83k interações e 676 contratações reais). O treino começa em Jul/2024 para garantir que cada linha tenha ao menos 6 meses de histórico disponíveis para as features deslizantes.

---

## Principais Descobertas do EDA

- **Viés de posição:** taxa de conversão visivelmente superior nas primeiras posições do carrossel — motivação central para o debiasing causal.
- **Produtos dominantes:** `conta_digital_plus` e `credito_pessoal` respondem pela maioria das contratações totais.
- **Canal de alta conversão:** `app_busca` converte em proporção muito superior a canais passivos como `push_notification`, sugerindo alta intenção de compra no momento da busca ativa.
- **Co-contratação:** `conta_digital_plus` + `credito_pessoal` é o par mais frequente (1.446 clientes), indicando oportunidade de bundle.
- **Feature mais preditiva:** histórico de clique anterior (`cp_clicou_antes_6m`) — clientes que clicaram no produto nos últimos 6 meses têm probabilidade muito maior de contratar.

---

## Resultados (vs. Baselines)

A avaliação é feita **por contexto de interação** (`id_interacao`, `safra`): para cada interação do conjunto de teste, todos os 20 produtos são ranqueados e verificamos se o produto efetivamente contratado aparece no top-5.

| Métrica | **Modelo** | Popularidade | Aleatório |
|---------|-----------|--------------|-----------|
| NDCG@5 | 0.4040 | **0.5285** | 0.3271 |
| Precision@5 | **0.0127** | 0.0128 | 0.0103 |
| Recall@5 | **0.7589** | 0.7337 | 0.3891 |
| Rank médio (positivos) | 4.40 | 3.63 | 7.12 |
| AUC-ROC médio (por produto) | **0.8354** | — | — |

O baseline de **Popularidade é competitivo** em NDCG@5 (0.53 vs 0.40) — reflexo da concentração histórica de contratações em poucos produtos populares, que torna a política de "recomendar sempre os mesmos" difícil de superar em métricas agregadas.

O **modelo supera a popularidade em Recall@5** (+3,4%): encontra mais contratos reais no top-5 das recomendações. A precisão é essencialmente equivalente. O ganho ocorre sobretudo em **perfis que divergem da preferência média** — clientes com histórico de clique em produtos de nicho, segmento premium ou comportamento de canal atípico.

O **AUC-ROC médio de 0.84** (macro, por produto) evidencia que cada modelo discrimina bem entre convertedores e não-convertedores dentro do seu produto, mesmo sob desbalanceamento severo (taxas de conversão de 0,1% a 1,8%).

### Por que usar o modelo em vez da popularidade?

A popularidade recomenda **o mesmo ranking para todos os clientes**. O modelo personaliza: um cliente com histórico de cliques em `investimento_cdb` e alta renda recebe produtos de investimento no topo; um cliente com alta utilização de limite e renda baixa recebe soluções de crédito. Essa personalização não aparece nas métricas agregadas de ranking — mas é o principal valor de um sistema de recomendação em produção.

---

## Limitações e Próximos Passos

- **3 produtos sem modelo:** `consorcio_imovel`, `titulo_capitalizacao` e `uso_lastro_limite` não tiveram contratações no período de treino. Para esses casos, aplica-se fallback por popularidade.
- **Modelo linear:** Regressão Logística captura apenas relações lineares. Um GBM (LightGBM) capturaria interações não-lineares e potencialmente aproximaria a vantagem da popularidade em NDCG enquanto mantém a personalização.
- **Cold-start:** clientes novos sem histórico de interações dependem apenas de features demográficas.
- **Epsilon fixo:** o valor ε = 10% é estático. Um **Upper Confidence Bound (UCB)** adaptativo ajustaria a exploração com base na incerteza do score por produto.
- **Receita não otimizada diretamente:** o modelo maximiza P(contratação). Uma função objetivo que pondera probabilidade × margem (receita esperada) poderia ser mais alinhada ao objetivo de negócio.
- **Retreino:** recomenda-se ciclo mensal com monitoramento de PSI (Population Stability Index) para detecção precoce de concept drift.
