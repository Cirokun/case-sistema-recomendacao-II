# Report Executivo — Sistema de Recomendação de Produtos Financeiros

## Abordagem

Implementamos um **modelo de Regressão Logística com Causal Inference (position debiasing)** e **Epsilon-Greedy** para servir as recomendações. A escolha da Regressão Logística se justifica pela interpretabilidade, velocidade de treino e inferência em escala, e pelo fato de o problema ser predominantemente de *ranking por score*, onde a calibração de probabilidade importa mais do que a fronteira de decisão complexa.

**Causal Inference — Position Debiasing:** o EDA revelou que a taxa de conversão cai de **2,31% na posição 1** para **0,01% na posição 20** — um viés 230× causado apenas pela exposição. Para remover esse efeito, incluímos `posicao_exibicao` como feature durante o treino (permitindo ao modelo separar o efeito do produto do efeito da posição) e fixamos `posicao = 1` na inferência. Isso equivale ao operador **do-calculus** de Pearl: estimamos a probabilidade de contratação *se* todos os produtos fossem exibidos na melhor posição, comparando-os em pé de igualdade.

**Epsilon-Greedy (ε = 10%):** 90% das vezes o modelo serve o ranking ótimo (exploitation); 10% das vezes embaralha as posições 6–20, mantendo o top-5 intacto para não prejudicar a experiência. Isso garante dados diversos para o próximo retreino, reduzindo o cold-start de produtos menos expostos.

**Split temporal:** treino nas safras 202401–202509 (88% dos dados, ~441k interações); teste nas safras 202510–202512 (12%, ~61k interações e 515 contratações reais).

---

## Principais Descobertas do EDA

- **Viés de posição:** conversão 230× maior na posição 1 vs. posição 20 — principal motivação para causal inference.
- **Produtos dominantes:** `conta_digital_plus` e `credito_pessoal` respondem por ~57% das contratações totais.
- **Canal de alta conversão:** `app_busca` converte 4,3% — 10× mais que `push_notification` (0,1%), sugerindo alta intenção de compra.
- **Co-contratação:** `conta_digital_plus` + `credito_pessoal` é o par mais frequente (1.446 clientes), indicando oportunidade de bundle.
- **Feature mais preditiva:** histórico de clique anterior (`cp_clicou_antes`) — clientes que clicaram antes têm chance muito maior de contratar.

---

## Resultados (vs. Baselines)

| Métrica | **Modelo** | Popularidade | Aleatório |
|---------|-----------|--------------|-----------|
| NDCG@5 | **0.9640** | 0.5493 | 0.1332 |
| Precision@5 | **0.2020** | — | — |
| Hit Rate@5 | **1.0000** | 0.7800 | 0.2333 |
| AUC-ROC | **0.9963** | — | — |
| Avg Precision | **0.6921** | — | — |

O modelo supera o baseline de popularidade em **+75% em NDCG@5** e **+28% em Hit Rate@5**, com Hit Rate@5 = 100% — para todos os 300 clientes avaliados com contratações reais no período de teste, ao menos um dos produtos contratados estava no top-5 recomendado.

---

## Limitações e Próximos Passos

- **Modelo linear:** Regressão Logística captura apenas relações lineares. Um GBM (LightGBM/XGBoost) ou modelo de aprendizado a partir de pares (LambdaMART) capturaria interações não-lineares e potencialmente melhoraria os rankings.
- **Cold-start:** clientes novos sem histórico de interações dependem apenas de features demográficas. Solução: embedding colaborativo ou matrix factorization para inicialização.
- **Epsilon fixo:** o valor ε = 10% é estático. Um **Upper Confidence Bound (UCB)** adaptativo ajustaria a exploração com base na incerteza do score por produto.
- **Receita não otimizada diretamente:** o modelo maximiza P(contratação). Uma função objetivo que pondera probabilidade × margem (receita esperada) poderia ser mais alinhada ao objetivo de negócio.
- **Retreino:** recomenda-se ciclo mensal com monitoramento de PSI (Population Stability Index) para detecção precoce de concept drift.
