# Loteca 8S-6D-0T — Estratégia 10-5-5

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

6. Exibir telemetria suficiente para auditar probabilidades base/calibradas, rankings, `risk_rank`, gaps, entropia, secos, duplos, fronteira 6º/7º, evidência histórica, impacto decisório, robustez, Hard/Soft Constraints e decomposição de `P(>=13)`.

---

# Hard Constraints

A aposta final só é válida se satisfizer simultaneamente:

```text
8 secos
6 duplos
0 triplos

10 Top1
5 Top2
5 Top3
```

Como existem 8 secos e 6 duplos:

```text
8 x 1 + 6 x 2 = 20 marcações
```

Logo:

```text
10 Top1 + 5 Top2 + 5 Top3 = 20 marcações
```

A contagem refere-se às marcações efetivamente presentes no volante.

## Flamengo

Quando o **FLAMENGO/RJ** participar do concurso, o resultado correspondente à sua vitória deve obrigatoriamente estar entre as marcações, independentemente de ocupar Top1, Top2 ou Top3.

---

# Soft Constraints

1. Favorecer ordenações que antecipem e concentrem Top1, especialmente nas 10 primeiras posições, privilegiando runs longas e baixa fragmentação.
2. Favorecer soluções que excluam a vitória do **PALMEIRAS/SP**, priorizando empate ou derrota quando isso não comprometer significativamente a qualidade global.
3. Soft Constraints nunca podem relaxar Hard Constraints.
4. O custo de uma Soft Constraint deve ser mensurável e exibido quando relevante.

---

# Hipótese estrutural 10-5-5

A estrutura 8S-6D-0T contém 20 marcações e precisa distribuir essas marcações exatamente em:

```text
10 Top1
5 Top2
5 Top3
```

Diferentemente da estratégia 9S-5D-0T — 9-5-5, não é possível usar simplesmente os 6 menores `p(top1)` como `Top2+Top3`, porque isso produziria:

```text
8 Top1
6 Top2
6 Top3
```

violando as Hard Constraints.

Uma baseline estrutural natural e válida é:

```text
8 secos Top1
4 duplos Top2+Top3
1 duplo Top1+Top2
1 duplo Top1+Top3
```

Essa configuração produz exatamente:

```text
Top1 = 8 + 1 + 1 = 10
Top2 = 4 + 1     = 5
Top3 = 4 + 1     = 5
```

Portanto:

```text
8 secos
6 duplos
0 triplos
10 Top1
5 Top2
5 Top3
20 marcações
```

Essa baseline não é regra obrigatória. É apenas uma referência estrutural que qualquer abordagem mais sofisticada precisa superar fora da amostra.

## Relação entre tipos de duplo

Definir:

```text
D12 = duplos Top1+Top2
D13 = duplos Top1+Top3
D23 = duplos Top2+Top3
```

Como existem 6 duplos:

```text
D12 + D13 + D23 = 6
```

As quantidades de secos por rank são determinadas por:

```text
SecoTop1 = 10 - D12 - D13
SecoTop2 = 5  - D12 - D23
SecoTop3 = 5  - D13 - D23
```

As três quantidades devem ser inteiras e não negativas e, obrigatoriamente:

```text
SecoTop1 + SecoTop2 + SecoTop3 = 8
```

Para a baseline preferencial:

```text
D12 = 1
D13 = 1
D23 = 4

SecoTop1 = 8
SecoTop2 = 0
SecoTop3 = 0
```

## Cobertura dos tipos de duplo

Para cada partida:

```text
P(Top1+Top2) = p(top1) + p(top2)
P(Top1+Top3) = p(top1) + p(top3)
P(Top2+Top3) = p(top2) + p(top3) = 1 - p(top1)
```

O tipo de duplo deve ser decidido globalmente, considerando as Hard Constraints e o efeito sobre `P(acertos >= 13)`.

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

Como existem 6 duplos, as métricas de seleção devem dar atenção especial à fronteira 6º/7º.

## RiskRankPrecision@6

```text
RiskRankPrecision@6 = Top1_fail entre risk_ranks 1..6 / 6
```

## RiskRankRecall@6

```text
RiskRankRecall@6 = Top1_fail capturados por risk_ranks 1..6 / Top1_fail totais
```

## Curva cumulativa Recall@k

Calcular:

```text
Recall@1
Recall@2
...
Recall@14
```

Destacar especialmente:

```text
Recall@5
Recall@6
Recall@7
```

para medir a qualidade da ordenação na região em que o limite estrutural de 6 duplos passa a ser decisivo.

## RiskRankNDCG@6

Usar NDCG@6 para medir a qualidade da ordenação das falhas no topo do ranking.

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
Precision@6
Recall@6
CoverageFail
DoubleWasteRate
RiskRankPrecision@6
RiskRankRecall@6
RiskRankNDCG@6
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

# Impacto decisório

Separar obrigatoriamente três níveis:

```text
1. probabilidades mudaram
2. ranking / top-6 mudou
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
Top-6 de risco alterado
Conjunto dos 6 duplos alterado
Bilhete final alterado
Acertos alterados
13+ alterado
```

## ConditionalImpact

Avaliar o Challenger também somente nos concursos em que o bilhete mudou:

```text
n_changed_tickets
mean_hits_champion_changed
mean_hits_challenger_changed
12plus_champion_changed
12plus_challenger_changed
13plus_champion_changed
13plus_challenger_changed
```

## DecisionNetGain

```text
DecisionNetGain = soma(acertos_challenger - acertos_champion)
```

calculada apenas nos concursos com bilhete diferente.

Também reportar:

```text
DecisionWinRate
DecisionLossRate
DecisionTieRate
```

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
-> histórico/risk_rank atua somente na escolha estrutural dos 6 duplos
-> otimizador 8S-6D-0T / 10-5-5
```

Nesse modo:

```text
não alterar p(1/X/2)
não alterar Top1/Top2/Top3
histórico só influencia a decisão estrutural de quais jogos e quais tipos de duplo usar
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

---

# Fronteira do 6º vs 7º candidato a duplo

Como o bilhete possui exatamente 6 duplos, a fronteira estrutural principal passa a ser:

```text
6º candidato a duplo
vs
7º candidato a duplo
```

A telemetria deve mostrar, quando aplicável:

```text
risk_rank
jogo
p(top1)
1 - p(top1)
tipo de duplo proposto
P(>=13) com decisão original
P(>=13) após troca
DeltaP13+
```

A comparação não deve assumir que os 6 duplos são necessariamente `Top2+Top3`, pois as Hard Constraints 10-5-5 exigem uma composição global válida entre `D12`, `D13` e `D23`.

---

# Otimização direta de P(>=13)

Para cada jogo, definir a probabilidade coberta pelas marcações selecionadas.

Para seco:

```text
c_i = p(resultado selecionado)
```

Para duplo:

```text
c_i = p(resultado A) + p(resultado B)
```

Assumindo independência entre jogos para fins do cálculo operacional do bilhete, obter a distribuição exata do número de acertos por convolução dinâmica.

Objetivo:

```text
P(>=13) = P(13) + P(14)
```

A otimização deve procurar diretamente o melhor bilhete entre todas as soluções que satisfaçam:

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

Soft Constraints só atuam como desempate ou preferência dentro do espaço válido e nunca podem reduzir silenciosamente a prioridade de `P(>=13)`.

---

# Validação independente das Hard Constraints

Depois da otimização, validar novamente o bilhete por uma rotina independente.

A execução deve falhar explicitamente se qualquer condição não for satisfeita:

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

Nunca corrigir silenciosamente um bilhete inválido após a otimização.

---

# Telemetria mínima da aposta final

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
risk_rank
pTop1_base
pTop1_ajustado
delta_pTop1
ranking_mudou
seco / duplo
palpite
ranks selecionados
probabilidade coberta
```

Resumo final:

```text
Secos: 8/8
Duplos: 6/6
Triplos: 0/0
Top1: 10/10
Top2: 5/5
Top3: 5/5
Marcações: 20/20
Flamengo: regra satisfeita
```

Decomposição probabilística:

```text
P(14)
P(13)
P(>=13)
P(12)
P(>=12)
```

---

# Critério de promoção Champion/Challenger

Uma nova técnica não deve ser promovida apenas porque melhora uma métrica intermediária.

Prioridade de decisão:

```text
1. Hard Constraints sempre satisfeitas
2. melhora real de >=13 em walk-forward
3. Net13Gain positivo
4. ausência de regressão relevante em >=12
5. DecisionNetGain / DecisionWinRate
6. calibração probabilística
7. robustez temporal
8. simplicidade e auditabilidade
```

Pequenos ganhos de log-loss, Brier ou ECE não justificam alterar o Champion quando não se convertem em decisões melhores no bilhete.

---

# Princípio geral

O projeto não procura simplesmente prever corretamente cada partida isoladamente.

Ele procura construir **um único bilhete de 14 jogos**, com exatamente **8 secos, 6 duplos, 0 triplos e distribuição 10-5-5**, cuja combinação de probabilidades, histórico e estrutura maximize a chance de chegar à faixa de premiação principal:

```text
P(acertos >= 13)
```

Toda melhoria deve ser demonstrada fora da amostra e sempre dentro das Hard Constraints.