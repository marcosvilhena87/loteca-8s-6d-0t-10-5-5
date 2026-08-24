# Loteca 8S-6D-0T — Estratégia 10-5-5

Projeto para geração de **um único palpite final por concurso da Loteca**, usando probabilidades históricas, calibração e otimização combinatória para maximizar prioritariamente:

```text
P(acertos >= 13)
```

A filosofia do projeto é simples: **qualquer técnica nova deve provar ganho fora da amostra e nunca pode violar as Hard Constraints**. Métricas como log-loss, Brier, ECE, média de acertos e estabilidade são evidências auxiliares; o objetivo final continua sendo a qualidade real do bilhete, especialmente na faixa de 13+.

---

# Estado atual da implementação

O pipeline atual executa:

```text
histórico
-> calibração por temperatura
-> validação cronológica
-> calibração opcional por rank
-> calibração por risk_rank
-> validação real Champion/Challenger dos bilhetes
-> otimização exata de P(>=13)
-> aplicação de Hard Constraints
-> aplicação de Soft Constraints dentro de faixa quase ótima
-> telemetria e auditorias estruturais
```

Componentes já implementados:

- calibração por temperatura com promoção somente quando melhora o holdout cronológico;
- calibração por rank Top1/Top2/Top3 com gate de validação;
- `risk_rank` de 1 a 14;
- shrinkage do `risk_rank` por tamanho de amostra e estabilidade temporal;
- IC95% para taxa observada de acerto do Top1 por `risk_rank`;
- validação real Champion/Challenger em bilhetes históricos;
- funil de impacto decisório;
- `DecisionNetGain`, win/loss/tie e média de acertos apenas nos bilhetes alterados;
- otimização global exata de `P(>=13)`;
- validação independente das Hard Constraints;
- matriz de substituições estruturais;
- fronteira diagnóstica do 6º/7º candidato a duplo;
- preferências anti-Palmeiras/Vasco dentro de uma faixa quase ótima;
- regra obrigatória de inclusão da vitória do Flamengo.

O `risk_rank` só é promovido quando, no holdout cronológico:

```text
1. melhora o log-loss
2. não reduz a quantidade de bilhetes 13+
3. Net13Gain >= 0
4. DecisionNetGain >= 0
```

Depois de aprovado, seus parâmetros são reestimados usando todo o histórico disponível antes da implantação.

---

# Hard Constraints

```text
14 jogos
8 secos
6 duplos
0 triplos
10 Top1
5 Top2
5 Top3
20 marcações
```

Em caso de empate de probabilidades, usar:

```text
1 > 2 > X
```

Quando o **FLAMENGO/RJ** participar, sua vitória deve obrigatoriamente estar entre as marcações.

Soft Constraints nunca podem relaxar Hard Constraints.

---

# Soft Constraints — preferências de solução

As Soft Constraints atuam somente depois de garantida a validade estrutural do bilhete e nunca podem violar as Hard Constraints.

## Preferência contra vitórias de Palmeiras e Vasco

Quando **PALMEIRAS/SP** ou **VASCO DA GAMA/RJ** participarem do concurso, favorecer soluções que **excluam a vitória dessas equipes**, priorizando empate ou derrota, desde que isso não comprometa significativamente a qualidade global da aposta.

A regra deve ser tratada como preferência, e não como proibição absoluta:

```text
1. encontrar a melhor P(>=13) possível dentro das Hard Constraints
2. definir uma faixa de soluções quase ótimas
3. dentro dessa faixa, favorecer a exclusão das vitórias de Palmeiras e Vasco
4. priorizar soluções que excluam a vitória de ambas as equipes
5. somente depois aplicar os demais critérios de desempate estrutural
```

A tolerância atualmente usada como referência é de **0,5% de perda relativa máxima** em `P(>=13)` em relação ao ótimo global.

Exemplo:

```text
P13plus_otimo = 0,04000
P13plus_candidato = 0,03985
perda_relativa = 0,375%
```

Nesse caso, o candidato permanece dentro da tolerância e pode ser preferido se excluir uma ou ambas as vitórias indesejadas.

Não adicionar bônus arbitrário diretamente a `P(>=13)`.

A ordem correta é:

```text
ótimo probabilístico
-> faixa aceitável de quase ótimos
-> preferência anti-Palmeiras/Vasco
-> demais critérios de desempate
```

---

# Estrutura 10-5-5

Definir:

```text
D12 = Top1+Top2
D13 = Top1+Top3
D23 = Top2+Top3
D12 + D13 + D23 = 6
```

Os secos por rank são:

```text
SecoTop1 = 10 - D12 - D13
SecoTop2 = 5  - D12 - D23
SecoTop3 = 5  - D13 - D23
```

Uma baseline estrutural válida é:

```text
8 secos Top1
4 D23
1 D12
1 D13
```

Ela produz exatamente 10 Top1, 5 Top2 e 5 Top3. Essa composição é apenas referência de pesquisa, não regra obrigatória.

---

# Cobertura dos duplos

```text
CoberturaD12 = p(top1) + p(top2)
CoberturaD13 = p(top1) + p(top3)
CoberturaD23 = p(top2) + p(top3) = 1 - p(top1)
```

Como `p(top1) >= p(top2) >= p(top3)`, isoladamente:

```text
CoberturaD12 >= CoberturaD13 >= CoberturaD23
```

A escolha final, porém, é global e deve fechar exatamente 10/5/5.

---

# DoubleGain e RecoveryGain

D12 e D13 preservam Top1 e acrescentam uma segunda marcação:

```text
DoubleGain(D12) = p(top2)
DoubleGain(D13) = p(top3)
```

D23 é diferente: ele abandona Top1 e troca por Top2+Top3.

```text
RecoveryGain(D23)
= CoberturaD23 - p(top1)
= 1 - 2*p(top1)
```

Não tratar D23 como `DoubleGain`.

Telemetria por jogo:

```text
p(top1)
p(top2)
p(top3)
CoberturaD12
CoberturaD13
CoberturaD23
tipo escolhido
DoubleGain, se D12/D13
RecoveryGain, se D23
```

---

# GameUncertainty e DoubleValue

Separar dois conceitos diferentes:

## GameUncertainty

Mede quão incerto é o jogo, independentemente da decisão estrutural.

Sugestão:

```text
GameUncertainty = Entropia / log(3)
```

## DoubleValue

Mede quanto a melhor cobertura de duas marcações acrescenta em relação ao seco Top1.

```text
DoubleValue = max(CoberturaD12, CoberturaD13, CoberturaD23) - p(top1)
```

Um jogo pode ser muito incerto sem necessariamente ser o melhor local para gastar um dos seis duplos, porque o fechamento global 10/5/5 pode tornar outra alocação superior.

---

# risk_rank

Ordenar as 14 partidas do maior risco relativo de falha do Top1 para o menor:

```text
risk_rank = 1..14
risk_rank=1  -> maior risco
risk_rank=14 -> menor risco
```

A calibração por `risk_rank` usa somente concursos anteriores, é validada cronologicamente e só é promovida quando há ganho fora da amostra.

A auditoria atual inclui:

```text
n
pTop1_medio_previsto
Top1_hit_observado
Top1_fail_observado
IC95% hit
RiskRankStability
HistoricalConfidence
lift_shrunk
```

O fator final sofre shrinkage de acordo com tamanho de amostra e estabilidade temporal, evitando transformar ruído histórico em grandes correções probabilísticas.

Métricas futuras úteis:

```text
RiskRankPrecision@6
RiskRankRecall@6
RiskRankNDCG@6
RiskRankECE
Brier por risk_rank
```

---

# Funil de impacto decisório

Uma melhora probabilística não basta. O projeto mede se a calibração chega de fato à aposta final.

Relatório:

```text
concursos avaliados
ranking mudou
duplos mudaram
bilhete final mudou
acertos mudaram
faixa 13+ mudou
```

Nos bilhetes alterados, comparar:

```text
média de acertos Champion -> Challenger
DecisionNetGain
DecisionWinRate
DecisionLossRate
DecisionTieRate
```

Isso permite distinguir:

```text
melhora de calibração
!=
melhora operacional do bilhete
```

---

# Otimização direta

Para cada jogo:

```text
seco:  c_i = p(resultado selecionado)
duplo: c_i = p(resultado A) + p(resultado B)
```

A distribuição exata de acertos é obtida por convolução dinâmica, sem Monte Carlo.

O objetivo é otimizar diretamente:

```text
P(>=13) = P(13) + P(14)
```

A solução final deve satisfazer:

```text
8 secos
6 duplos
0 triplos
10 Top1
5 Top2
5 Top3
20 marcações
regra obrigatória do Flamengo
```

---

# Decomposição do objetivo

Para cada bilhete final, reportar:

```text
P(14)
P(13)
P(>=13)
P(12)
P(>=12)
```

Também auditar a igualdade entre o valor calculado pela distribuição dinâmica e o valor mantido pelo otimizador.

---

# Validação independente

Após a otimização, validar novamente:

```text
jogos = 14
secos = 8
duplos = 6
triplos = 0
Top1 = 10
Top2 = 5
Top3 = 5
marcações = 20
Flamengo = regra satisfeita, quando aplicável
```

Nunca corrigir silenciosamente um bilhete inválido.

---

# Matriz de substituições estruturais

Para cada duplo selecionado, testar substituições estruturalmente válidas e recalcular exatamente `P(>=13)`.

Telemetria:

```text
DuploOriginal
JogoSubstituto
TipoOriginal
TipoSubstituto
P13plus_original
P13plus_alternativo
DeltaP13plus
```

Definir:

```text
DeltaP13plus = P13plus_alternativo - P13plus_original
```

Essa auditoria mede o custo real de deslocar um duplo, respeitando a estrutura 10/5/5.

---

# P13+ Regret por decisão — prioridade alta

Próximo aprimoramento recomendado: generalizar a matriz de substituições para qualquer decisão relevante do bilhete.

Definir:

```text
P13plusRegret(decisão)
= P13plus_ótimo
- P13plus_melhor_bilhete_que_força_decisão_alternativa
```

Objetivo: responder quanto custa contrariar cada decisão do otimizador.

Exemplo de telemetria:

```text
Jogo
Decisão atual
Melhor alternativa estrutural válida
P13plus atual
P13plus alternativo
Regret
Classificação de fragilidade
```

Classificação sugerida:

```text
regret muito baixo -> decisão de fronteira
regret intermediário -> decisão moderadamente robusta
regret alto -> decisão estruturalmente robusta
```

O `P13+ Regret` deve ser preferido a conclusões baseadas somente em `pTop1` ou `risk_rank`.

---

# Auditoria histórica da fronteira 6º × 7º — prioridade alta

A fronteira simples por `1-p(top1)` é apenas diagnóstico, pois o fechamento 10/5/5 pode impedir uma troca direta.

Mesmo assim, concursos em que o 6º e o 7º candidatos ficam muito próximos merecem auditoria histórica.

Faixas sugeridas:

```text
margem <= 0.01
margem <= 0.02
margem <= 0.05
```

Para esses casos, comparar se a decisão Champion ou a melhor alternativa estrutural teria produzido mais acertos.

Features candidatas para desempate futuro:

```text
p(top1)
gap12
gap13
entropia
GameUncertainty
DoubleValue
RecoveryGain
risk_rank
HistoricalConfidence
tipo de duplo necessário
P13plusRegret
```

Nenhum `cutoff_score` deve ser implantado antes de validação Champion/Challenger fora da amostra.

---

# Robustez do bilhete a perturbações — prioridade alta

O bilhete ótimo pode ser sensível a pequenas mudanças probabilísticas.

Testar perturbações controladas, por exemplo:

```text
±0,5 ponto percentual
±1,0 ponto percentual
±2,0 pontos percentuais
```

Após cada perturbação, renormalizar as probabilidades e reotimizar.

Métricas sugeridas:

```text
TicketStability@0.5pp
TicketStability@1.0pp
TicketStability@2.0pp
DecisionStability por jogo
DoubleSetStability
Top1SetStability
```

Objetivo:

```text
distinguir decisão ótima de decisão robustamente ótima
```

---

# Auditoria histórica D12 / D13 / D23 — prioridade alta

Como D23 elimina completamente o Top1, ele deve ser auditado separadamente.

Por tipo:

```text
n
cobertura prevista
cobertura observada
acertos
lift
```

Para D23, medir adicionalmente:

```text
Top1 acertou
Top2/Top3 acertou
D23 evitou erro do seco
D23 causou erro ao excluir Top1
NetRecoveryGain
RecoveryRate
```

Isso permite validar se a estratégia de abandonar o favorito nominal funciona de forma consistente fora da amostra.

---

# Explicação de secos inesperados

Quando um jogo de `risk_rank` baixo permanecer seco enquanto outro de rank posterior receber duplo, emitir diagnóstico estrutural.

Exemplo:

```text
Jogo A permaneceu seco porque promovê-lo a duplo exigiria
compensação estrutural que reduziria P(>=13) em X.
```

Sempre que possível, quantificar com `DeltaP13plus` ou `P13plusRegret`.

---

# Relatório “por que este duplo?”

A telemetria deve evoluir para uma explicação automática por decisão.

Exemplo conceitual:

```text
Jogo 10
✓ pTop1 baixo
✓ alto risco de falha do seco
✓ forte RecoveryGain
✓ duplo estruturalmente robusto
✓ alto P13plusRegret se removido

Classificação: DUPLO MUITO ROBUSTO
```

E, para uma decisão de fronteira:

```text
✓ duplo melhora cobertura
⚠ último duplo selecionado
⚠ alternativa muito próxima
⚠ baixo P13plusRegret

Classificação: DUPLO DE FRONTEIRA
```

---

# Backtest walk-forward multi-janelas — prioridade máxima

Além do holdout cronológico 80/20, implementar validação expanding-window.

Exemplo:

```text
Treino 1..200 -> Teste 201..220
Treino 1..220 -> Teste 221..240
Treino 1..240 -> Teste 241..260
...
```

Comparar no mínimo:

```text
A = probabilidades brutas
B = + temperatura
C = + temperatura + risk_rank
D = Champion atual
E = Challenger em avaliação
```

Relatório mínimo por janela e agregado:

```text
Concursos
LogLoss
Brier
14
>=13
>=12
mean_hits
median_hits
Net13Gain
DecisionNetGain
DecisionWinRate
bilhetes alterados
```

O ganho estimado de `P(>=13)` pelo próprio modelo não prova ganho real.

A promoção de novos mecanismos deve privilegiar consistência entre várias janelas, e não desempenho excepcional em um único holdout.

---

# Bootstrap pareado e incerteza — prioridade alta

Como `>=13` é raro, diferenças pequenas podem ser ruído.

Para Champion/Challenger, estimar por reamostragem pareada de concursos:

```text
IC95% do delta de mean_hits
IC95% do DecisionNetGain
IC95% do Net13Gain
P(DecisionNetGain > 0)
P(Net13Gain > 0)
```

O bootstrap deve preservar o pareamento Champion/Challenger por concurso.

Inicialmente, essas métricas devem funcionar como diagnóstico, não necessariamente como Hard Gate de promoção.

---

# Champion / Challenger formal

Toda nova regra, modelo ou heurística deve entrar primeiro como Challenger.

```text
Champion = versão atualmente implantada
Challenger = nova técnica candidata
```

Relatório padrão:

```text
Métrica                  Champion   Challenger   Delta
LogLoss
Brier
mean_hits
12+
13+
14
Net13Gain
DecisionNetGain
DecisionWinRate
bilhetes alterados
```

Uma mudança só deve ser promovida quando:

```text
1. todas as Hard Constraints forem satisfeitas
2. houver melhora ou não inferioridade real em 13+
3. Net13Gain não for negativo
4. não houver regressão relevante em 12+
5. DecisionNetGain for compatível
6. houver robustez temporal
7. não houver evidência forte de sobreajuste
```

Log-loss, Brier, ECE e `mean_hits` são evidências auxiliares.

---

# Backtest por composição D12/D13/D23

Registrar por concurso:

```text
D12
D13
D23
hits
14
13+
12+
P13_estimado
DoubleWaste
RecoverySuccess
```

Consolidar por composição:

```text
D12 D13 D23 | concursos | 14 | 13+ | 12+ | mean_hits
```

Também reportar:

```text
Net13Gain
DecisionNetGain
RecoveryRate
DoubleWasteRate
```

Não promover uma composição fixa apenas porque ela aparece com frequência no otimizador.

---

# Robustez temporal das composições

Comparar:

```text
últimos 50 concursos
últimos 100 concursos
últimos 200 concursos
histórico completo
```

Sinais instáveis devem sofrer shrinkage ou permanecer apenas como diagnóstico.

---

# Ablation estrutural

Comparar:

```text
A = otimizador livre atual
B = composição fixa 4 D23 + 1 D12 + 1 D13
C = composição aprendida historicamente
D = composição dinâmica por risk_rank + gaps + entropia
```

Critério principal:

```text
>=13
Net13Gain
```

---

# StructuralCost

Medir apenas para auditoria:

```text
P13plus_relaxado = melhor P(>=13) com 8 secos e 6 duplos sem impor 10/5/5
P13plus_10_5_5  = melhor P(>=13) respeitando 10/5/5
StructuralCost  = P13plus_relaxado - P13plus_10_5_5
```

`StructuralCost` nunca autoriza relaxar Hard Constraints.

---

# Pesquisa condicional Top1_fail -> Top2 / Top3

Estudar:

```text
P(top2_hit | top1_fail)
P(top3_hit | top1_fail)
```

Treinar um Challenger somente nos jogos históricos com:

```text
top1_hit = 0
```

Target sugerido:

```text
Top2 -> 1
Top3 -> 0
```

Features candidatas:

```text
risk_rank
gap12
gap13
gap23
entropia
p(top1)
p(top2)
p(top3)
posição no concurso
janelas históricas
```

A aplicação inicial preferencial é orientar D12/D13/D23, sem substituir probabilidades base sem validação.

---

# Possível evolução futura do risk_rank

Não adicionar novos conjuntos de fatores categóricos enquanto o ganho atual ainda estiver sendo consolidado.

Uma evolução futura possível é substituir os 14 fatores independentes por uma função suavizada:

```text
f(risk_rank)
```

ou:

```text
f(pTop1, risk_percentile)
```

Isso pode reduzir descontinuidades artificiais entre ranks vizinhos.

Só testar depois que o pipeline de walk-forward, bootstrap e Champion/Challenger estiver consolidado.

---

# Telemetria mínima

Para cada jogo:

```text
Jogo
Mandante x Visitante
p(1)
p(X)
p(2)
top1 / top2 / top3
p(top1) / p(top2) / p(top3)
gap12
gap13
entropia
GameUncertainty
risk_rank
pTop1_base
pTop1_ajustado
delta_pTop1
ranking_mudou
CoberturaD12
CoberturaD13
CoberturaD23
DoubleValue
seco / duplo
palpite
ranks selecionados
probabilidade coberta
DoubleGain ou RecoveryGain
P13plusRegret, quando disponível
DecisionStability, quando disponível
```

Resumo:

```text
Secos: 8/8
Duplos: 6/6
Triplos: 0/0
Top1: 10/10
Top2: 5/5
Top3: 5/5
Marcações: 20/20
Composição D12/D13/D23
Flamengo: regra satisfeita
```

---

# Roadmap priorizado

## Prioridade 1

```text
P13+ Regret por jogo/decisão
Auditoria histórica da fronteira 6º x 7º
Walk-forward multi-janelas
```

## Prioridade 2

```text
Bootstrap pareado do ganho
Robustez a perturbações probabilísticas
Auditoria histórica D12/D13/D23
Champion/Challenger padronizado
```

## Prioridade 3

```text
GameUncertainty x DoubleValue
Relatório automático “por que este duplo?”
Pesquisa Top1_fail -> Top2/Top3
```

## Prioridade futura

```text
risk_rank suavizado
modelos mais complexos somente após evidência clara de necessidade
```

---

# O que evitar por enquanto

Evitar adicionar complexidade sem evidência fora da amostra:

```text
redes neurais apenas por sofisticação
XGBoost/LightGBM sem baseline convincente
calibrações excessivamente segmentadas
14x3 ou mais parâmetros adicionais sem shrinkage
bônus manual a zebras
pesos arbitrários de entropia
novas Soft Constraints sem backtest
Monte Carlo onde a solução exata já existe
```

A vantagem atual do projeto é combinar **transparência, cálculo exato e validação cronológica**. Essa característica deve ser preservada.

---

# Princípio geral

O projeto procura construir **um único bilhete de 14 jogos**, com exatamente **8 secos, 6 duplos, 0 triplos e distribuição 10-5-5**, cuja combinação de probabilidades, histórico e estrutura maximize:

```text
P(acertos >= 13)
```

A evolução do sistema deve seguir esta ordem:

```text
medir
-> auditar
-> criar Challenger
-> validar fora da amostra
-> medir incerteza
-> promover somente se houver evidência
```

**Toda melhoria deve ser demonstrada fora da amostra e sempre dentro das Hard Constraints.**
