# Loteca 9S-5D-0T — Estratégia 9-5-5

Projeto para geração de **um único palpite final por concurso da Loteca**, usando `data/concursos_anteriores.csv` e as informações do próximo concurso para maximizar prioritariamente:

```text
P(acertos >= 13)
```

O projeto deve respeitar integralmente as Hard Constraints. Probabilidades, calibrações, histórico, meta-modelos, consenso, heurísticas e Soft Constraints só podem atuar dentro do espaço de soluções válidas.

---

# Estratégia

1. Gerar um único palpite final por concurso.
2. Produzir `p(1)`, `p(X)` e `p(2)` para cada partida.
3. Ordenar os três resultados em `top1`, `top2` e `top3`.
4. Em empate de probabilidades, usar obrigatoriamente:

```text
1 > 2 > X
```

5. Representar o resultado real por One-Hot Encoding:

```text
top1_hit
top2_hit
top3_hit
```

Em cada partida, exatamente uma dessas variáveis deve ser igual a `1`.

6. Exibir telemetria suficiente para auditar probabilidades base/calibradas, rankings, `risk_rank`, gaps, entropia, secos, duplos, fronteira 5º/6º, evidência histórica, impacto decisório, robustez, Hard/Soft Constraints e decomposição de `P(>=13)`.

---

# Hard Constraints

A aposta final só é válida se satisfizer simultaneamente:

```text
9 secos
5 duplos
0 triplos

9 Top1
5 Top2
5 Top3
```

Como existem 9 secos e 5 duplos:

```text
9 x 1 + 5 x 2 = 19 marcações
```

Logo:

```text
9 Top1 + 5 Top2 + 5 Top3 = 19 marcações
```

A contagem refere-se às marcações efetivamente presentes no volante.

## Flamengo

Quando o **FLAMENGO/RJ** participar do concurso, o resultado correspondente à sua vitória deve obrigatoriamente estar entre as marcações, independentemente de ocupar Top1, Top2 ou Top3.

---

# Soft Constraints

1. Favorecer ordenações que antecipem e concentrem Top1, especialmente nas 9 primeiras posições, privilegiando runs longas e baixa fragmentação.
2. Favorecer soluções que excluam a vitória do **PALMEIRAS/SP**, priorizando empate ou derrota quando isso não comprometer significativamente a qualidade global.
3. Soft Constraints nunca podem relaxar Hard Constraints.
4. O custo de uma Soft Constraint deve ser mensurável e exibido quando relevante.

---

# Hipótese estrutural 9-5-5

Baseline estrutural:

```text
9 maiores p(top1) -> seco Top1
5 menores p(top1) -> duplo Top2+Top3
```

Essa configuração satisfaz automaticamente 9 secos, 5 duplos, 9 Top1, 5 Top2 e 5 Top3.

Para um jogo `Top2+Top3`:

```text
P(cobertura) = 1 - p(top1)
```

Essa baseline não é regra obrigatória. É a referência que qualquer abordagem mais sofisticada precisa superar fora da amostra.

## Relação entre tipos de duplo

```text
D12 = duplos Top1+Top2
D13 = duplos Top1+Top3
D23 = duplos Top2+Top3

D12 + D13 + D23 = 5

SecoTop1 = 4 + D23
SecoTop2 = D13
SecoTop3 = D12
```

Portanto:

- `Top1+Top2` exige um Top3 seco em outra partida;
- `Top1+Top3` exige um Top2 seco em outra partida;
- `Top2+Top3` permite manter um Top1 seco adicional.

---

# Histórico como segunda fonte de decisão

O histórico não deve servir apenas para treinar probabilidades. Ele deve fornecer evidência explícita para decisões estruturais.

Princípio preferencial:

```text
presente define o contexto
histórico resolve a dúvida
```

Estudar historicamente:

```text
top1_hit
top2_hit
top3_hit
risk_rank
```

incluindo frequência global/recente, distribuição por concurso, runs, fragmentação, transições Top1/Top2/Top3, posição, tipo de ranking, `risk_rank`, concursos 13+, concursos `RECOVERABLE`, zona de cutoff e estabilidade temporal.

---

# Calibração probabilística

## Temperatura

Promover apenas quando melhorar log-loss em validação cronologicamente posterior.

```text
Temperatura candidata
Temperatura implantada
Log-loss bruto
Log-loss calibrado
Status: promovida / rejeitada
```

## Calibração global por rank Top1/Top2/Top3

Pode ser testada, mas só altera a implantação se melhorar a validação fora da amostra.

Se rejeitada:

```text
lifts Top1/Top2/Top3 = [1.0, 1.0, 1.0]
```

Uma calibração global rejeitada não implica que sinais históricos relativos ao concurso sejam inúteis.

---

# `risk_rank`: prioridade atual de pesquisa

Para cada concurso, ordenar as 14 partidas do maior risco relativo de falha do Top1 para o menor:

```text
risk_rank = 1..14

risk_rank=1  -> maior risco relativo
risk_rank=14 -> menor risco relativo
```

O `risk_rank` reduz dependência da magnitude crua ao transformar confiança em **posição relativa dentro do próprio concurso**.

## Calibração histórica por `risk_rank`

O treinamento pode estimar, para cada `risk_rank`, um fator suavizado entre frequência observada e probabilidade prevista de acerto do Top1.

Regras obrigatórias:

1. usar somente concursos cronologicamente anteriores;
2. preservar a proporção relativa de Top2/Top3 quando ajustar Top1;
3. avaliar em bloco posterior fora da amostra;
4. promover somente se houver ganho de validação;
5. usar fatores neutros `1.0` quando rejeitada;
6. nunca relaxar Hard Constraints.

Telemetria:

```text
Calibração por risk_rank: promovida / rejeitada
Log-loss risk_rank: base=... calibrado=...
```

---

# Auditoria histórica do `risk_rank`

## Base vs ajustado por jogo

```text
Jogo
risk_rank
pTop1_base
pTop1_ajustado
delta_pTop1
top1_base
top1_ajustado
ranking_mudou?
```

Registrar também alterações de Top2/Top3.

## Tabela histórica 1..14

```text
risk_rank
n
pTop1_medio_previsto
Top1_hit_observado
Top1_fail_observado
CalibrationError
lift_hit
lift_fail
IC95%
RiskRankStability
HistoricalConfidence
```

Definir:

```text
CalibrationError = Top1_hit_observado - pTop1_medio_previsto
```

Destacar os ranks com maior `overconfidence` e `underconfidence`.

## Intervalos de confiança

Para cada rank, reportar `n`, hit rate, fail rate e IC95%. A força do ajuste deve cair quando a evidência for mais incerta.

## Shrinkage

```text
lift_shrunk = 1 + alpha * (lift - 1)
0 <= alpha <= 1
```

`alpha` pode depender de amostra, largura do IC, estabilidade temporal e consistência entre janelas.

## Estabilidade temporal

Comparar:

```text
últimos 50
últimos 100
últimos 200
histórico completo
```

Criar `RiskRankStability` e reduzir o peso de sinais instáveis.

## Monotonicidade / isotonic calibration

Testar como Challenger uma curva monotônica em que:

```text
Top1_fail(rank1) >= Top1_fail(rank2) >= ... >= Top1_fail(rank14)
```

Usar isotonic regression somente se melhorar o desempenho walk-forward.

---

# Métricas próprias do `risk_rank`

## RiskRankPrecision@5

```text
RiskRankPrecision@5 = Top1_fail entre risk_ranks 1..5 / 5
```

## RiskRankRecall@5

```text
RiskRankRecall@5 = Top1_fail capturados por risk_ranks 1..5 / Top1_fail totais
```

## Curva cumulativa Recall@k

Calcular:

```text
Recall@1
Recall@2
...
Recall@14
```

Destacar especialmente `Recall@5`, `Recall@6` e `Recall@7` para medir quanto da dificuldade vem do seletor e quanto vem do limite estrutural de apenas 5 duplos.

## RiskRankNDCG@5

Usar NDCG@5 para medir a qualidade da ordenação das falhas no topo do ranking.

## RiskRankECE

Criar ECE específico por `risk_rank`:

```text
RiskRankECE = soma ponderada |observado - previsto|
```

## Brier por `risk_rank`

Adicionar Brier Score para complementar o log-loss e evitar promoção baseada em uma única métrica probabilística.

---

# Backtest real BASE vs RISK_RANK

O ganho de `P(>=13)` calculado com probabilidades ajustadas é **estimado pelo próprio modelo**. Ele não prova sozinho ganho real.

Executar walk-forward estrito comparando:

```text
A = probabilidades brutas
B = + temperatura
C = + temperatura + risk_rank
```

Relatório mínimo:

```text
Modelo
Concursos
LogLoss
Brier
RiskRankECE
14
>=13
>=12
mean_hits
median_hits
RecoveryRate
Precision@5
Recall@5
CoverageFail
DoubleWasteRate
RiskRankPrecision@5
RiskRankRecall@5
RiskRankNDCG@5
```

A contribuição incremental do `risk_rank` deve ser medida nos resultados reais, não apenas em `P13+` calculado.

---

# Net13Gain

```text
Net13Gain =
  concursos que passaram de <13 para >=13
- concursos que passaram de >=13 para <13
```

Reportar separadamente:

```text
11 -> 13
12 -> 13
12 -> 14
13 -> 12
13 -> 11
14 -> 13
14 -> <13
```

Essa métrica tem prioridade maior que pequenas diferenças de log-loss.

---

# Matriz de transição de acertos

Comparar os acertos concurso a concurso antes/depois do componente histórico.

```text
BASE\RISK | 10 | 11 | 12 | 13 | 14
10        | .. | .. | .. | .. | ..
11        | .. | .. | .. | .. | ..
12        | .. | .. | .. | .. | ..
13        | .. | .. | .. | .. | ..
14        | .. | .. | .. | .. | ..
```

A matriz deve mostrar se o componente está recuperando `12->13`, melhorando `11->12`, ou apenas deslocando resultados sem ganho na cauda.

---

# Impacto decisório — prioridade atual

A partir do momento em que a calibração por `risk_rank` melhora log-loss, mas não demonstra ganho claro no bilhete, o foco passa a ser medir **quando o histórico realmente muda uma decisão** e se essa intervenção é benéfica.

Separar obrigatoriamente três níveis:

```text
1. probabilidades mudaram
2. ranking / top-5 mudou
3. bilhete final mudou
```

Somente o terceiro nível pode alterar diretamente o número real de acertos daquele concurso.

## TicketChangeRate

```text
TicketChangeRate =
concursos em que o bilhete final do Challenger difere do Champion
/
concursos avaliados
```

Também calcular:

```text
Top1RankingChangeRate
DoubleSetChangeRate
FinalTicketChangeRate
```

## Funil de impacto

Produzir um relatório sequencial:

```text
Concursos avaliados
Probabilidades alteradas
Algum ranking 1/X/2 alterado
Top-5 de risco alterado
Conjunto dos 5 duplos alterado
Bilhete final alterado
Acertos alterados
13+ alterado
```

Esse funil deve permitir distinguir melhora de calibração de melhora operacional.

## ConditionalImpact

Avaliar o Challenger também **somente nos concursos em que o bilhete mudou**:

```text
n_changed_tickets
mean_hits_champion_changed
mean_hits_challenger_changed
12plus_champion_changed
12plus_challenger_changed
13plus_champion_changed
13plus_challenger_changed
```

A avaliação global continua obrigatória; `ConditionalImpact` é diagnóstico complementar.

## DecisionNetGain

```text
DecisionNetGain =
soma(acertos_challenger - acertos_champion)
```

calculada apenas nos concursos com bilhete diferente.

Também reportar:

```text
DecisionWinRate
DecisionLossRate
DecisionTieRate
```

## NetPrizeTierGain

Além de `Net13Gain`, registrar migrações de faixa:

```text
<12 -> 12
<13 -> 13
<14 -> 14
13 -> 12
14 -> 13
14 -> <13
```

O objetivo principal continua sendo `>=13`, mas a decomposição ajuda a entender onde as intervenções ganham ou perdem valor.

---

# RISK_CALIBRATION vs RISK_SELECTOR

O histórico pode agregar valor em dois lugares diferentes e eles devem ser testados separadamente.

## `RISK_CALIBRATION`

```text
probabilidades -> ajuste histórico por risk_rank -> ranking -> otimizador
```

Pode alterar `p(1)`, `p(X)`, `p(2)` e eventualmente Top1/Top2/Top3.

## `RISK_SELECTOR_ONLY`

```text
TEMP
-> p(1), p(X), p(2) preservadas
-> Top1/Top2/Top3 preservados
-> histórico/risk_rank atua somente na escolha dos 5 duplos
-> otimizador 9-5-5
```

Nesse modo:

```text
não alterar p(1/X/2)
não alterar Top1/Top2/Top3
histórico só influencia a decisão estrutural de quais jogos recebem duplo
```

## Ablation obrigatória

Comparar:

```text
A = TEMP_ONLY
B = TEMP + RISK_CALIBRATION
C = TEMP + RISK_SELECTOR_ONLY
D = TEMP + RISK_CALIBRATION + RISK_SELECTOR
```

Métricas mínimas:

```text
>=13
>=12
Net13Gain
mean_hits
TicketChangeRate
DecisionNetGain
DecisionWinRate
RecoveryRate
CutoffDecisionAccuracy
```

Hipótese central:

> o `risk_rank` pode ter pouco valor como recalibrador de probabilidades, mas valor relevante como seletor de decisões marginais.

---

# RISK_RANK_ONLY baseline

Manter a baseline ordinal mais simples possível:

```text
risk_rank 1..5  -> duplo Top2+Top3
risk_rank 6..14 -> seco Top1
```

Ela deve ser comparada ao `RISK_SELECTOR_ONLY` para medir quanto ganho vem apenas da ordenação relativa e quanto depende de histórico adicional.

---

# `HistoricalRiskScore`

Separar conceitualmente:

```text
pTop1_rank = posição atual do jogo segundo p(top1)
HistoricalRiskScore = evidência histórica de vulnerabilidade do Top1
```

Exemplo:

```text
HistoricalRiskScore =
    w1 * historical_fail_by_risk_rank
  + w2 * RiskRankStability
  + w3 * cutoff_history
  + w4 * ranking_type_history
  + w5 * recoverable_history
```

O score pode ser usado apenas para ordenar candidatos da zona cinzenta, sem ser convertido em nova probabilidade.

Pesos somente por walk-forward.

---

# Zona cinzenta adaptativa

Não assumir obrigatoriamente uma zona fixa 4..8.

Testar pelo menos duas definições:

```text
ORDINAL:
risk_rank 4..8

DISTANCE_TO_CUTOFF:
|pTop1_i - pTop1_cutoff| <= threshold
```

O threshold deve ser escolhido somente no treino/validação.

A zona cinzenta deve concentrar a capacidade do histórico; núcleo de duplos e núcleo de secos muito robustos não devem ser reabertos sem evidência excepcional.

---

# Cutoff histórico 5º vs 6º

Para cada concurso histórico, armazenar:

```text
candidate_rank5
candidate_rank6
pTop1_rank5
pTop1_rank6
margin_56
gap12_5
gap12_6
entropy_5
entropy_6
ranking_type_5
ranking_type_6
top1_hit_rank5
top1_hit_rank6
```

Classificar:

```text
A = rank5 fail / rank6 hit   -> rank5 correto
B = rank5 hit / rank6 fail   -> rank6 deveria entrar
C = ambos fail               -> ambos úteis
D = ambos hit                -> ambos desperdício
```

## CutoffDecisionAccuracy

Calcular somente nos casos A/B:

```text
CutoffDecisionAccuracy = decisões corretas / casos discriminantes
```

Não diluir com C/D.

---

# CutoffDecisionDataset

Criar dataset específico das decisões próximas ao cutoff:

```text
contest_id
candidate_rank5
candidate_rank6
candidate_rank7
margin_56
margin_57
gap12_rank5/6/7
entropy_rank5/6/7
ranking_type_rank5/6/7
historical_fail_rank5/6/7
RiskRankStability_rank5/6/7
HistoricalConfidence_rank5/6/7
resultado_real_rank5/6/7
```

Targets possíveis:

```text
best_double_candidate
KEEP_5 vs SWAP_6
KEEP_5 vs SWAP_7
```

---

# SwapOpportunityRate

Medir quantas oportunidades reais de melhora existiam na fronteira.

```text
SwapOpportunityRate =
concursos em que trocar um duplo de cutoff por um seco da zona cinzenta aumentaria acertos
/
concursos avaliados
```

E:

```text
SwapCapturedRate =
oportunidades de swap corretamente capturadas pelo seletor
/
oportunidades de swap existentes
```

Separar análise 5->6, 5->7 e demais swaps permitidos.

---

# MinimalRecoverySwap

Para cada concurso abaixo de 13, especialmente os de 12 acertos, identificar a menor alteração válida capaz de melhorar o resultado.

Registrar:

```text
n_swaps_minimos
duplo_que_sai
seco_que_entra
risk_rank de ambos
margin
gap_rank
entropy_rank
ranking_type
HistoricalRiskScore
HistoricalConfidence
```

Priorizar casos:

```text
12 -> 13
12 -> 14
11 -> 13
```

---

# RECOVERY_SELECTOR

Criar um Challenger especializado em concursos `RECOVERABLE`.

Treino retrospectivo permitido:

```text
modelo < 13
oracle_9_5_5 >= 13
```

Objetivo:

```text
identificar qual decisão estrutural de duplo impediu 13+
```

O modelo só pode usar features disponíveis ex ante no momento do concurso.

## RecoveryPrecision

```text
RecoveryPrecision =
trocas sugeridas que realmente aumentariam acertos
/
trocas sugeridas
```

## RecoveryRecall

```text
RecoveryRecall =
oportunidades recuperáveis capturadas
/
oportunidades recuperáveis totais
```

Essas métricas complementam `RecoveryRate`.

---

# DoNoHarmGate

Como o histórico pode alterar um bilhete que já é probabilisticamente forte, qualquer intervenção deve ter um gate explícito.

Exemplo conceitual:

```text
swap permitido somente se:
HistoricalConfidence >= threshold_confidence
AND CutoffDecisionScore >= threshold_decision
AND Delta_P13 >= -tolerancia
```

Os thresholds e a tolerância devem ser aprendidos/definidos apenas em validação.

O gate não pode ser usado para justificar intervenção com base no concurso alvo.

## Penalidade por intervenção

Testar seletor conservador que prefira **não alterar** o Champion quando a evidência histórica for fraca.

Pode-se adicionar custo de intervenção:

```text
DecisionScore_final = DecisionScore_historico - lambda * intervention_cost
```

`lambda` somente por walk-forward.

---

# Consenso para swap

Para trocar rank5 por rank6/rank7, permitir votos independentes:

```text
gap12
a entropia
historical_fail
cutoff_history
pairwise_model
HistoricalConfidence
RECOVERY_SELECTOR
```

Exemplo:

```text
swap se >= k votos
```

O valor de `k` deve ser validado fora da amostra.

Manter telemetria de cada voto para auditoria.

---

# Pareto da zona cinzenta

Antes de aplicar score ponderado, testar dominância de Pareto.

Um candidato pode ser preferido quando domina outro em vários sinais relevantes, por exemplo:

```text
menor pTop1
menor gap12
maior entropy
maior historical_fail
maior HistoricalConfidence
```

Se nenhum domina, ambos permanecem no Pareto front e o desempate pode ser feito pelo histórico/pairwise.

Essa abordagem reduz dependência de pesos arbitrários.

---

# Decision Stability

Além de `Stability@5`, medir estabilidade da decisão marginal:

```text
CutoffStability = frequência com que o mesmo candidato permanece na vaga de cutoff sob perturbações plausíveis
```

Também registrar, por candidato da zona cinzenta:

```text
SelectionFrequency
```

em bootstrap/Monte Carlo de probabilidades.

Exemplo:

```text
J6 78%
J3 49%
J8 31%
```

---

# RankingChangeImpact

Quando o ajuste histórico alterar a identidade de Top1/Top2/Top3, registrar:

```text
n_ranking_changes
changes_correct
changes_wrong
net_hit_gain_from_changes
ranking_change_success_rate
```

Pergunta principal:

> quando o histórico muda o ranking concreto, a mudança produz mais acertos do que erros?

---

# RISK_RANK_FREE vs RISK_RANK_PRESERVE_ORDER

Testar:

```text
RISK_RANK_FREE
- pode alterar a ordem 1/X/2

RISK_RANK_PRESERVE_ORDER
- ajusta confiança
- preserva Top1/Top2/Top3 originais
```

Comparar:

```text
>=13
>=12
Net13Gain
TicketChangeRate
DecisionNetGain
mean_hits
RankingChangeImpact
LogLoss
Brier
```

---

# Oracle específico do ranking de risco

Diagnóstico retrospectivo:

```text
quantos concursos RECOVERABLE exigiam sair do top-5 risk_rank?
quantas falhas estavam em risk_rank 6/7/8?
quantas eram recuperáveis com uma única troca?
```

Separar:

```text
limitação do ranking de risco
vs
limitação estrutural dos 5 duplos
```

Nunca usar oracle para prever o próximo concurso.

---

# Pairwise 5 vs 6

Treinar Challenger especializado na decisão crítica do cutoff.

Entrada:

```text
delta_pTop1
delta_gap12
delta_entropy
ranking_type_5
ranking_type_6
CalibrationError_5
CalibrationError_6
RiskRankStability_5
RiskRankStability_6
HistoricalConfidence_5
HistoricalConfidence_6
```

Saída:

```text
KEEP_5
SWAP_6
```

Métricas:

```text
CutoffDecisionAccuracy
DecisionNetGain
Net13Gain
RecoveryPrecision
RecoveryRecall
```

---

# Modelo da zona cinzenta

Somente depois de validar o pairwise 5/6, expandir para candidatos próximos ao cutoff.

Target:

```text
top1_fail
```

Features preferenciais:

```text
risk_rank
rank_gap12
rank_entropy
ranking_type
posição
CalibrationErrorByRiskRank
RiskRankStability
HistoricalConfidence
HistoricalRiskScore
regime do concurso
```

Preferir features ordinais/categóricas quando tiverem desempenho equivalente ou superior às magnitudes contínuas.

---

# HistoricalConfidence operacional

```text
HistoricalConfidence = f(
  tamanho_da_amostra,
  largura_IC,
  estabilidade_temporal,
  consistência_entre_fontes
)
```

Pode modular a intensidade de intervenção:

```text
adjustment_strength = base_adjustment * HistoricalConfidence
```

Pesos somente por walk-forward.

---

# Estratégias menos dependentes das probabilidades cruas

## `PROBABILITY_ONLY`

```text
score_duplo = 1 - p(top1)
```

## `RANK_ONLY`

Usa probabilidades apenas para formar Top1/Top2/Top3 e posições ordinais.

## `RISK_RANK_ONLY`

```text
risk_rank 1..5  -> duplo Top2+Top3
risk_rank 6..14 -> seco Top1
```

## `RISK_SELECTOR_ONLY`

Preserva probabilidades e ranking concreto, usando o histórico somente para decidir os cinco jogos que recebem duplo.

## `HISTORICAL_ONLY`

Challenger diagnóstico usando apenas evidências históricas permitidas temporalmente.

## `HYBRID_ORDINAL`

Pode usar:

```text
risk_rank
rank_gap12
rank_gap13
rank_entropy
HistoricalRiskScore
HistoricalConfidence
```

O objetivo é preservar ou melhorar `>=13` reduzindo dependência da magnitude crua.

---

# HistoricalScore e HistoricalVote

## HistoricalScore

```text
HistoricalScore =
    w1 * historical_risk_rank
  + w2 * historical_ranking_type
  + w3 * historical_position
  + w4 * historical_knn_game
  + w5 * historical_regime
  + w6 * historical_recency
```

Preferir ranks, percentis e scores normalizados. Pesos somente por walk-forward.

## HistoricalVote

```text
H1 = risk_rank
H2 = tipo de ranking
H3 = posição
H4 = KNN por jogo
H5 = KNN por concurso
H6 = recência/estabilidade
H7 = pattern matching
H8 = RECOVERABLE / 12->13
```

Manter voto histórico separado do probabilístico na telemetria.

---

# KNN histórico

## Por jogo

Features preferenciais:

```text
tipo_de_ranking
risk_rank
pTop1_percentile
gap12_percentile
gap13_percentile
entropy_percentile
posição
```

Exibir `n_vizinhos`, Top1_fail, Top1_hit, voto KNN e distância média.

## Por concurso

Assinatura global:

```text
mean_pTop1
median_pTop1
mean_entropy
mean_gap12
n_top1_home
n_top1_draw
n_top1_away
n_jogos_equilibrados
n_favoritos_fortes
quantidade de cada tipo de ranking
```

Analisar quantidade de Top1_fail, distribuição por `risk_rank`, runs e falhas próximas ao cutoff.

---

# Núcleo e zona cinzenta

Classificar os jogos em:

```text
núcleo_duplo
zona_cinzenta
núcleo_seco
```

Filosofia:

```text
probabilidade / estrutura -> define núcleo
histórico                 -> resolve zona cinzenta
```

---

# Oracle histórico 9-5-5

Implementar `oracle_9_5_5` exclusivamente para diagnóstico retrospectivo.

```text
SUCCESS
RECOVERABLE
UNRECOVERABLE
```

- `SUCCESS`: estratégia real atingiu 13+;
- `RECOVERABLE`: ficou abaixo de 13, mas existia aposta válida 9-5-5 capaz de atingir 13+;
- `UNRECOVERABLE`: nenhuma aposta válida conseguiria 13+.

```text
RecoveryRate = SUCCESS / (SUCCESS + RECOVERABLE)
```

Nunca usar oracle para prever o próximo concurso.

---

# Análise 12 -> 13

Para concursos com exatamente 12 acertos, identificar a menor alteração necessária para chegar a 13.

```text
% recuperável com 1 troca
% recuperável com 2 trocas
% recuperável com 3+ trocas
```

Criar dataset quando:

```text
acertos_modelo = 12
acertos_oracle >= 13
```

Registrar duplo que deveria sair, seco que deveria entrar, `risk_rank`, margin, ranking type, gap/entropia ordinal, HistoricalRiskScore e HistoricalConfidence.

---

# Métricas específicas dos duplos

```text
Precision@5 = falhas de Top1 entre os 5 duplos / 5
Recall@5 = falhas de Top1 capturadas pelos duplos / falhas Top1 totais
CoverageFail = Recall@5
DoubleWasteRate = duplos em que Top1_hit=1 / 5
```

Objetivo auxiliar:

```text
maximizar CoverageFail
minimizar DoubleWasteRate
```

---

# Fronteira 5º vs 6º

Exibir:

```text
Rank | Jogo | pTop1 | 1-pTop1 | risk_rank | Decisão
```

Calcular:

```text
P13+ original
P13+ após troca
Delta absoluto
Delta relativo
Margem pTop1
HistoricalRiskScore 5º/6º
HistoricalConfidence 5º/6º
CutoffDecision signal
DoNoHarmGate status
```

Separar:

```text
Fronteira probabilística
Evidência histórica
Robustez no objetivo
Impacto decisório
```

---

# Função objetivo

```text
max P(acertos >= 13)
```

Exibir:

```text
P(14)
P(13)
P(>=13)
P(12)
P(>=12)
```

Sob hipótese de independência entre jogos, a distribuição pode ser calculada exatamente por programação dinâmica / Poisson-binomial.

**Importante:** `P(>=13)` calculado com probabilidades ajustadas é estimativa interna. Promoção depende do desempenho real walk-forward.

---

# Matriz de trocas

```text
Sai | Entra | Delta P13+ | Delta HistoricalRiskScore | Delta Consensus | Gate
```

A matriz deve destacar oportunidades de swap da zona cinzenta e registrar se a intervenção seria permitida pelo `DoNoHarmGate`.

---

# Robustez

## Stability@5

```text
Stability@5 = persistência média dos 5 duplos
```

## Rank Preservation Stress Test

Preservar Top1/Top2/Top3, comprimir/expandir magnitudes, renormalizar, rerodar e medir `Agreement@5`/`Stability@5`.

## Temperature Stress Test

Testar diagnosticamente:

```text
T = 0.70
T = 0.90
T = 1.00
T = 1.20
T = 1.50
```

## Decision Stability

Medir `CutoffStability` e frequência de seleção dos candidatos da zona cinzenta sob perturbações plausíveis.

---

# Walk-forward e prevenção de leakage

Fluxo obrigatório:

```text
treina até concurso N-1
calibra usando apenas dados <= N-1
calcula risk_rank/estatísticas usando apenas dados permitidos
calcula HistoricalRiskScore somente com concursos <= N-1
ajusta thresholds/gates somente com dados <= N-1
busca KNN somente em concursos <= N-1
prevê concurso N
monta aposta N
avalia resultado real N
avança para N+1
```

Nenhuma estatística histórica, threshold de swap, gate, score ou modelo de recuperação pode incorporar direta ou indiretamente o resultado do concurso avaliado.

---

# Validação por período

Reportar:

```text
últimos 50 concursos
51-100 anteriores
101-200 anteriores
histórico completo
```

Uma estratégia só é robusta se o ganho não depender de um único período.

---

# Bootstrap de 13+

```text
hit_rate_13plus = ...
IC95% = [... ; ...]
```

Também aplicar bootstrap ao delta de `hit_rate_13plus`, `Net13Gain` e, quando houver amostra suficiente, `DecisionNetGain` entre Champion e Challenger.

---

# Promotion Gate

Hierarquia para promover qualquer intervenção histórica/seletor:

```text
1. hit_rate_13plus melhora ou fica estatisticamente equivalente
2. Net13Gain >= 0 e preferencialmente > 0
3. hit_rate_12plus não piora materialmente
4. DecisionNetGain >= 0
5. DecisionWinRate > DecisionLossRate quando houver amostra suficiente
6. RecoveryRate / RecoveryPrecision / RecoveryRecall apresentam sinal útil
7. CutoffDecisionAccuracy apresenta sinal útil
8. TicketChangeRate não é alto sem benefício correspondente
9. estabilidade por período
10. Brier / ECE / LogLoss não pioram materialmente
```

Melhorar log-loss isoladamente não é suficiente.

---

# Ablation study

Comparar explicitamente:

```text
A - probabilidades brutas
B - TEMP_ONLY
C - TEMP + RISK_CALIBRATION
D - RISK_RANK_PRESERVE_ORDER
E - RISK_RANK_FREE
F - RISK_RANK_ONLY
G - RISK_SELECTOR_ONLY
H - RISK_CALIBRATION + RISK_SELECTOR
I - + HistoricalRiskScore
J - + CutoffDecisionDataset / pairwise 5-6
K - + RECOVERY_SELECTOR
L - + DoNoHarmGate
M - + consenso de swap
N - + Pareto gray zone
O - + KNN histórico
P - + Soft Constraint Palmeiras
Q - modelo completo
```

Relatório mínimo:

```text
Modelo
Concursos
>=13
>=12
Net13Gain
Média
TicketChangeRate
DoubleSetChangeRate
DecisionNetGain
DecisionWinRate
DecisionLossRate
RecoveryRate
RecoveryPrecision
RecoveryRecall
Precision@5
Recall@5
RiskRankPrecision@5
RiskRankRecall@5
CutoffDecisionAccuracy
SwapOpportunityRate
SwapCapturedRate
Stability@5
LogLoss
Brier
```

---

# Champion / Challenger

```text
Champion = estratégia atualmente aprovada
Challenger = nova implementação em avaliação
```

Challengers:

```text
RISK_RANK_ONLY
RISK_CALIBRATION
RISK_SELECTOR_ONLY
RISK_CALIBRATION_PLUS_SELECTOR
RISK_RANK_PRESERVE_ORDER
RISK_RANK_FREE
PAIRWISE_5_6
PAIRWISE_GRAY_ZONE
RECOVERY_SELECTOR
HISTORY_DECISION_ONLY
HYBRID_ORDINAL
DO_NO_HARM_HISTORY
PARETO_GRAY_ZONE
CONSENSUS_SWAP
KNN_HISTORY
```

Promoção somente por walk-forward.

---

# Hierarquia de métricas

```text
1. hit_rate_13plus
2. Net13Gain
3. hit_rate_12plus
4. DecisionNetGain
5. RecoveryRate
6. RecoveryPrecision / RecoveryRecall
7. Precision@5 / Recall@5 / CoverageFail
8. CutoffDecisionAccuracy
9. SwapCapturedRate
10. RiskRankPrecision@5 / RiskRankRecall@5
11. mean_hits
12. TicketChangeRate / DecisionStability
13. robustez / Stability@5
14. Brier / RiskRankECE / LogLoss
```

O produto final é a aposta completa, não o estimador probabilístico isolado.

---

# Guardrail contra overfitting

Nenhuma complexidade adicional entra na estratégia principal sem ganho fora da amostra sobre uma baseline mais simples.

Critérios:

- walk-forward estrito;
- amostra suficiente;
- melhoria em `>=13` ou equivalência com ganho estrutural comprovado;
- `Net13Gain` não negativo;
- `DecisionNetGain` não negativo;
- ausência de deterioração material em `>=12`;
- estabilidade por período;
- resultado não dependente de poucos concursos extremos;
- bootstrap/IC compatível com ganho plausível;
- penalidade por intervenção quando apropriado;
- nenhuma informação futura em features, gates, thresholds ou calibrações.

---

# Telemetria esperada

## Calibração

```text
Temperatura candidata/implantada
LogLoss base/calibrado
Calibração Top1/2/3: promovida/rejeitada
Calibração risk_rank: promovida/rejeitada
LogLoss risk_rank
Brier risk_rank
RiskRankECE
```

## Auditoria histórica `risk_rank`

```text
Rank | n | pTop1 previsto | hit observado | fail observado | erro calibração | IC95% | estabilidade | confiança | lift
```

## Impacto decisório

```text
Concursos avaliados
Probabilidades alteradas
Rankings alterados
Top-5 alterados
Conjuntos de duplos alterados
Bilhetes finais alterados
TicketChangeRate
DoubleSetChangeRate
DecisionWinRate
DecisionLossRate
DecisionTieRate
DecisionNetGain
Net13Gain
```

## Por partida

```text
Jogo
Mandante x Visitante
p_base(1/X/2)
p_ajustado(1/X/2)
top1/top2/top3 antes/depois
ranking_mudou?
risk_rank
gap12
gap13
entropy
rank_gap12
rank_entropy
HistoricalRiskScore
HistoricalConfidence
CutoffDecisionScore
grupo: núcleo_duplo / zona_cinzenta / núcleo_seco
tipo: seco ou duplo
palpite
cobertura
motivo da escolha
```

## Backtest comparativo

```text
TEMP_ONLY
RISK_CALIBRATION
RISK_SELECTOR_ONLY
RISK_CALIBRATION + RISK_SELECTOR

>=13
>=12
Net13Gain
TicketChangeRate
DecisionNetGain
RecoveryRate
CutoffDecisionAccuracy
```

## Fronteira

```text
5º candidato
6º candidato
7º candidato
margin_56
margin_57
P13+ original/após troca
delta absoluto/relativo
HistoricalRiskScore
HistoricalConfidence
CutoffDecisionScore
DoNoHarmGate
SwapOpportunity
fronteira probabilística
evidência histórica
robustez no objetivo
```

## Validação final

```text
Secos: 9/9
Duplos: 5/5
Triplos: 0/0
Top1: 9/9
Top2: 5/5
Top3: 5/5
Total: 19/19
Flamengo/RJ: regra satisfeita, quando aplicável
P(14)
P(13)
P(>=13)
P(12)
P(>=12)
```

A validação deve ser recalculada independentemente após a otimização e lançar erro se qualquer Hard Constraint for violada.

---

# Soft Constraint Palmeiras — custo explícito

```text
Melhor solução absoluta: P13+ = ...
Melhor solução sem vitória do Palmeiras: P13+ = ...
Custo da preferência: ...
```

A preferência só deve prevalecer quando o custo for aceitável.

---

# Estrutura do repositório

```text
.
├── main.py
├── data/
│   ├── concursos_anteriores.csv
│   └── proximo_concurso.csv
├── scripts/
│   ├── preprocess_data.py
│   ├── train_model.py
│   └── predict_results.py
└── output/
    └── predictions.csv
```

Responsabilidades:

- `main.py`: orquestração e telemetria;
- `data/concursos_anteriores.csv`: treinamento, calibração e backtest;
- `data/proximo_concurso.csv`: concurso alvo;
- `scripts/preprocess_data.py`: leitura, validação e features;
- `scripts/train_model.py`: treinamento, calibração, `risk_rank`, histórico, seletores e avaliação walk-forward;
- `scripts/predict_results.py`: probabilidades, rankings, seletores, evidência histórica, otimização 9-5-5 e palpite final;
- `output/predictions.csv`: saída auditável.

---

# Formato dos CSVs

```text
delimitador: ;
separador decimal das odds: ,
```

```python
pd.read_csv(caminho, sep=";", decimal=",")
```

---

# Convenção dos palpites

```text
Secos:    1 | X | 2
Duplos:   1X | 12 | X2
Triplos:  1X2
```

Configuração atual:

```text
0 triplos
```

---

# Execução

```bash
python main.py
```

Testes:

```bash
python -m unittest discover -v
```

Durante o treinamento, avaliações de bilhetes devem sempre usar o mesmo otimizador e o mesmo validador de Hard Constraints usados na previsão final. Comparações Champion/Challenger devem ser concurso a concurso e cronologicamente honestas.

---

# Ordem recomendada de implementação

```text
1. TicketChangeRate / DoubleSetChangeRate / Top1RankingChangeRate
2. funil de impacto decisório
3. ConditionalImpact
4. DecisionNetGain / DecisionWinRate / DecisionLossRate
5. TEMP_ONLY vs RISK_CALIBRATION vs RISK_SELECTOR_ONLY
6. RISK_RANK_ONLY baseline
7. CutoffDecisionDataset
8. CutoffDecisionAccuracy
9. SwapOpportunityRate / SwapCapturedRate
10. MinimalRecoverySwap
11. RECOVERY_SELECTOR
12. RecoveryPrecision / RecoveryRecall
13. DoNoHarmGate / penalidade por intervenção
14. consenso para swap
15. Pareto da zona cinzenta
16. Decision Stability / CutoffStability
17. pairwise 5 vs 6
18. modelo da zona cinzenta adaptativa
19. RISK_RANK_PRESERVE_ORDER vs FREE
20. HistoricalRiskScore / HistoricalConfidence
21. oracle específico do risk_rank
22. oracle_9_5_5 / RECOVERABLE / 12->13
23. KNN histórico
24. matriz de trocas
25. Stability@5 / stress tests
26. bootstrap do delta de >=13 e DecisionNetGain
27. Champion / Challenger
28. ablation study contínuo
```

A prioridade imediata é **descobrir se o histórico agrega valor quando atua como seletor de decisões marginais, e não apenas como recalibrador das probabilidades**.

---

# Perguntas experimentais centrais

> **Quantas vezes o histórico realmente muda o bilhete?**

> **Quando muda, a decisão melhora mais concursos do que piora?**

> **`RISK_SELECTOR_ONLY` supera `RISK_CALIBRATION` em `Net13Gain`, `DecisionNetGain` e recuperação 12->13?**

> **A decisão crítica entre o 5º, 6º e 7º candidatos pode ser melhorada historicamente sem overfitting?**

> **Quantas oportunidades reais de swap existem e quantas o seletor consegue capturar?**

> **Um `DoNoHarmGate` conservador melhora o saldo das intervenções históricas?**

> **Os concursos `RECOVERABLE` revelam padrões úteis para transformar 12 em 13?**

> **Quanto da qualidade atual pode ser preservado usando sinais relativos/ordinais em vez das probabilidades cruas?**

---

# Regras fundamentais

> **As Hard Constraints têm precedência absoluta sobre qualquer Soft Constraint, heurística, calibração ou preferência.**

> **A métrica decisiva é a qualidade da aposta completa para atingir pelo menos 13 acertos, validada fora da amostra.**

> **`P13+` estimado pelo próprio modelo não substitui `hit_rate_13plus` real em walk-forward.**

> **Melhorar log-loss é útil, mas não substitui melhora real de 13+.**

> **Uma intervenção histórica que não muda o bilhete não pode melhorar os acertos reais daquele concurso; por isso impacto decisório deve ser auditado separadamente da calibração.**

> **Valorizar o histórico só é melhoria se aumentar ou preservar o desempenho real de 13+ em walk-forward.**

> **Reduzir dependência das probabilidades só é melhoria se preservar ou aumentar o desempenho real de 13+.**