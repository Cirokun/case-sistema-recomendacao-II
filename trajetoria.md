# Trajetória na condução do case

Este arquivo fala sobre minha trajetória na condução do case, para compartilhar meu processo de pensamento, aprendizados e arrependimentos.

## Discovery

Começei o case bem entusiasmado, pois nunca havia aplicado um modelo de recomendação de sistemas antes.

Em minha primeira análise do problema, lembrei de uma técnica chamada **Thompson Sampling** que havia visto durante meu estágio e sabia que poderia ter algo a ver com o desafio de recomendar produtos financeiros. A ideia de explorar e exploitar opções com base em distribuições de probabilidade parecia se encaixar bem no contexto de um carrossel de produtos.

## Evolução das abordagens

### Thompson Sampling e a necessidade de contexto

Rapidamente percebi que o Thompson Sampling puro não era suficiente: faltava a adição de contexto do usuário na solução. O algoritmo tratava todos os usuários de forma homogênea, sem personalização, o que era claramente inadequado para um sistema de recomendação financeira.

### Contextual Bandits com Regressão Logística Bayesiana

Com esse diagnóstico, migrei para a abordagem de **Contextual Bandits**, que incorpora features dos usuários na tomada de decisão. Tentei implementá-la via regressão logística bayesiana, mas encontrei um problema estrutural: pela grande quantidade de observações no dataset, as variâncias das distribuições a priori dos coeficientes convergiam para valores minúsculos. Isso reduzia drasticamente o potencial de exploração do algoritmo, tornando-o excessivamente exploitativo. Isso também me gerou dúvidas mais profundas sobre como modelos de Thompson Sampling lidam com **data shift** ou **reward shift** ao longo do tempo — questões que ficaram em aberto.

### Learning to Rank

Pesquisando mais, vi que esse tipo de problema de ranqueamento de itens em um carrossel é tratado pela literatura como **Learning to Rank**. No entanto, os dados fornecidos não continham o carrossel inteiro por interação — apenas os itens com os quais o usuário interagiu — e percebi que isso parecia ser uma limitação importante para soluções de LTR, que geralmente dependem da ordem completa de exposição para modelar corretamente os feedbacks positivos e negativos.

### Solução final: Regressão por produto com debias de posição

Por fim, dada a restrição de tempo, optei por uma solução mais direta: **regressões individuais por produto**, com correção de viés de posição (*position debiasing*). O objetivo era isolar o que era sinal de contexto do usuário daquilo que era apenas viés da posição do item no carrossel — itens no topo naturalmente recebem mais cliques independentemente da relevância. Achei isso uma forma de introduzir Inferência Causal no case, um área de mais domínio meu.

## Reflexões sobre o processo

Gostaria muito de ter tido mais tempo para:

- **Organizar melhor o notebook**: a condução acelerada deixou o código menos estruturado do que eu gostaria.
- **Testar diferentes abordagens**: as hipóteses levantadas ao longo do processo mereciam experimentação mais sistemática.
- **Refatorar código repetitivo**: alguns trechos do notebook ficaram com duplicações que, com mais tempo, teria abstraído em funções reutilizáveis.

Utilizei bastante **LLMs** para ganhar velocidade na escrita do código, o que ajudou muito. No entanto, confesso que a forma como o código foi estruturado acabou dificultando alguns testes que eu gostaria de ter realizado com mais tempo disponível.

## Aprendizados

Apesar de todas as dificuldades, fiquei muito feliz de ter conduzido o case. Aprendi muito e fiquei super curioso sobre sistemas de recomendação e sobre como essa personalização — tão presente em qualquer aplicativo moderno — é tratada na prática pelas empresas e pelo próprio Itaú.

Gostaria muito de saber como outras pessoas lidaram com o mesmo problema, para discutir diferentes abordagens e continuar aprendendo sobre o assunto.
