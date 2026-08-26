# Loteca 8S-6D-0T — Estratégia 10-5-5

Projeto para geração de **um único palpite final por concurso da Loteca**, usando probabilidades históricas, calibração e otimização combinatória para maximizar prioritariamente:

```text
P(acertos >= 13)
```

A filosofia do projeto é simples: **qualquer técnica nova deve provar ganho fora da amostra e nunca pode violar as Hard Constraints**. Log-loss, Brier, ECE, média de acertos e estabilidade são evidências auxiliares; o objetivo final continua sendo a qualidade real do bilhete, especialmente na faixa de 13+.

---

# Estado atual da implementação

O pipeline atual executa:

```text
histórico
-> calibração por temperatura
-> validação cronológica
-> calibração opcional por rank
-> calibração por risk_rank
-> validação real Champion/Challenger
-> otimização exata de P(>=13)
-> aplicação de Hard Constraints
-> aplicação de Soft Constraints dentro de faixa quase ótima
-> validação independente
-> matriz de substituições estruturais
-> P13+ Regret por reotimização global
-> telemetria e auditorias estruturais
```

Componentes implementados:

- calibração por temperatura com promoção somente quando melhora o holdout cronológico;
- calibração por rank Top1/Top2/Top3 com gate de validação;
- `risk_rank` de 1 a 14;
- shrinkage do `risk_rank` por tamanho de amostra e estabilidade temporal;
- IC95% para taxa observada de acerto do Top1 por `risk_rank`;
- validação Champion/Challenger em bilhetes históricos;
- funil de impacto decisório;
- `DecisionNetGain`, win/loss/tie e média de acertos nos bilhetes alterados;
- otimização global exata de `P(>=13)`;
- validação independente das Hard Constraints;
- matriz de substituições estruturais;
- `P13plusRegret` por decisão com reotimização global;
- fronteira diagnóstica do 6º/7º candidato a duplo;
- preferência anti-Palmeiras/Vasco dentro de faixa quase ótima;
- regra obrigatória de inclusão da vitória do Flamengo.

---

# Snapshot de validação — 25/08/2026

```text
Calibração por temperatura: promovida
Temperatura: 0.88
Log-loss bruto:      0.966102
Log-loss calibrado: 0.964164

Calibração por rank: rejeitada
Calibração risk_rank: promovida
Log-loss antes do risk_rank: 0.964164
Log-loss após risk_rank:     0.964018
```

Validação real dos bilhetes no holdout:

```text
13+: 1 -> 1
12+: 4 -> 5
Net13Gain: 0
Média de acertos: 8.728 -> 8.761
```

Funil de impacto decisório:

```text
concursos avaliados: 92
ranking mudou:        15
conjunto de duplos:   15
bilhete final mudou:  32
acertos mudaram:      18
faixa 13+ mudou:       0
```

Nos 32 bilhetes alterados:

```text
acertos médios: 8.344 -> 8.438
DecisionNetGain: +3
DecisionWinRate:  34.4%
DecisionLossRate: 21.9%
DecisionTieRate:  43.8%
```

Interpretação atual: o `risk_rank` apresenta ganho pequeno de calibração, mas efeito operacional positivo, sem regressão observada em 13+. Ainda não existe evidência suficiente para afirmar aumento da frequência de 13+.

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

Em empate de probabilidades:

```text
1 > 2 > X
```

Quando o **FLAMENGO/RJ** participar, sua vitória deve obrigatoriamente estar entre as marcações.

Soft Constraints nunca podem relaxar Hard Constraints.

---

# Soft Constraints

## Preferência contra vitórias de Palmeiras e Vasco

Quando **PALMEIRAS/SP** ou **VASCO DA GAMA/RJ** participarem, favorecer soluções que excluam a vitória dessas equipes, desde que isso não comprometa significativamente `P(>=13)`.

Ordem correta:

```text
1. ótimo probabilístico dentro das Hard Constraints
2. faixa de soluções quase ótimas
3. preferência anti-Palmeiras/Vasco
4. demais critérios de desempate
```

Tolerância atual:

```text
perda relativa máxima em P(>=13) = 0,5%
```

Não adicionar bônus arbitrários diretamente ao objetivo.

---

# Estrutura 10-5-5

```text
D12 = Top1 + Top2
D13 = Top1 + Top3
D23 = Top2 + Top3
D12 + D13 + D23 = 6
```

Secos por rank:

```text
SecoTop1 = 10 - D12 - D13
SecoTop2 = 5  - D12 - D23
SecoTop3 = 5  - D13 - D23
```

Baseline estrutural válida:

```text
8 secos Top1
4 D23
1 D12
1 D13
```

Essa composição é apenas referência; o otimizador continua livre para escolher qualquer composição que satisfaça 10-5-5.

---

# Cobertura, DoubleGain e RecoveryGain

```text
CoberturaD12 = p(top1) + p(top2)
CoberturaD13 = p(top1) + p(top3)
CoberturaD23 = p(top2) + p(top3) = 1 - p(top1)
```

```text
DoubleGain(D12) = p(top2)
DoubleGain(D13) = p(top3)
```

D23 abandona Top1:

```text
RecoveryGain(D23)
= CoberturaD23 - p(top1)
= 1 - 2*p(top1)
```

Não tratar D23 como `DoubleGain`.

---

# GameUncertainty e DoubleValue

```text
GameUncertainty = Entropia / log(3)
```

Mede a incerteza intrínseca do jogo.

```text
DoubleValue = max(CoberturaD12, CoberturaD13, CoberturaD23) - p(top1)
```

Mede o ganho potencial da melhor cobertura dupla em relação ao seco Top1.

Um jogo pode ser muito incerto sem ser o melhor lugar para gastar um duplo, porque o fechamento global 10-5-5 pode tornar outra alocação superior.

---

# risk_rank

Ordenar as 14 partidas do maior risco relativo de falha do Top1 para o menor:

```text
risk_rank = 1..14
risk_rank=1  -> maior risco
risk_rank=14 -> menor risco
```

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

O fator final sofre shrinkage conforme tamanho de amostra e estabilidade temporal.

Próximas métricas formais:

```text
RiskRankPrecision@6
RiskRankRecall@6
RiskRankNDCG@6
RiskRankECE
RiskRankBrier
```

`Precision@6`: entre os seis jogos mais arriscados, quantos realmente falharam no Top1.

`Recall@6`: entre todas as falhas Top1 do concurso, quantas estavam concentradas nesses seis jogos.

---

# Funil de impacto decisório

Uma melhora probabilística não basta. O projeto mede se a alteração chega de fato ao bilhete final.

```text
concursos avaliados
ranking mudou
duplos mudaram
bilhete final mudou
acertos mudaram
faixa 13+ mudou
```

Nos bilhetes alterados:

```text
média de acertos Champion -> Challenger
DecisionNetGain
DecisionWinRate
DecisionLossRate
DecisionTieRate
```

Princípio:

```text
melhora de calibração != melhora operacional do bilhete
```

---

# Otimização direta

Para cada jogo:

```text
seco:  c_i = p(resultado selecionado)
duplo: c_i = p(resultado A) + p(resultado B)
```

A distribuição exata de acertos é calculada por convolução dinâmica, sem Monte Carlo.

Objetivo:

```text
P(>=13) = P(13) + P(14)
```

Decomposição reportada:

```text
P(14)
P(13)
P(>=13)
P(12)
P(>=12)
```

A igualdade entre a distribuição dinâmica e o valor mantido pelo otimizador deve ser auditada em toda execução.

---

# Validação independente

Após a otimização:

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

# Matriz de substituições estruturais — IMPLEMENTADA

Para cada duplo selecionado, testar substituições estruturalmente válidas e recalcular exatamente `P(>=13)`.

```text
DeltaP13plus = P13plus_alternativo - P13plus_original
```

A matriz mede trocas locais. Ela continua útil como diagnóstico, mas não substitui uma reotimização global condicionada.

---

# P13+ Regret por decisão — IMPLEMENTADO

```text
P13plusRegret(decisão)
= P13plus_ótimo
- P13plus_melhor_bilhete_condicionado_a_decisão_alternativa
```

A implementação atual proíbe, uma a uma, cada combinação escolhida e reotimiza globalmente os 14 jogos, preservando todas as Hard Constraints.

Telemetria:

```text
Jogo
Decisão atual
Melhor alternativa global
P13plus alternativo
Regret absoluto
Regret relativo
Classificação de robustez
```

O `P13plusRegret` deve ter precedência sobre conclusões baseadas somente em `pTop1`, `risk_rank` ou posição 6º/7º.

## Extensão recomendada: Regret por alternativa

Além da melhor alternativa global, calcular separadamente o custo de cada tipo de decisão possível:

```text
RegretSeco
RegretD12
RegretD13
RegretD23
```

Exemplo:

```text
J6 atual = D12
forçar seco -> regret X
forçar D13  -> regret Y
forçar D23  -> regret Z
```

Isso permite responder não apenas se uma decisão é robusta, mas **qual mudança seria a menos ruim**.

## Classificação granular sugerida

Usar preferencialmente o regret relativo:

```text
< 0,5%      -> FRONTEIRA
0,5%–1,5%  -> FRÁGIL
1,5%–3,0%  -> MODERADA
3,0%–7,0%  -> ROBUSTA
> 7,0%      -> MUITO ROBUSTA
```

Essas faixas devem começar como telemetria e podem ser ajustadas após auditoria histórica.

---

# Motor genérico de otimização condicionada — RECOMENDADO

Criar uma interface única para contrafactuais, por exemplo:

```text
optimize_with_constraint(...)
```

Capacidades desejadas:

```text
forbid_choice
force_choice
force_dry
force_double
force_contains_top1
force_excludes_top1
```

Objetivo: permitir que `P13plusRegret`, auditorias estruturais e futuros testes usem o mesmo motor, evitando versões paralelas do otimizador.

---

# Robustez a perturbações probabilísticas — PRIORIDADE 1

O ótimo matemático pode depender de diferenças probabilísticas pequenas.

Testar perturbações controladas:

```text
±0,5 ponto percentual
±1,0 ponto percentual
±2,0 pontos percentuais
```

Fluxo:

```text
perturbar probabilidades
-> renormalizar
-> reotimizar
-> comparar decisões com o ótimo original
```

Métricas:

```text
TicketStability@0.5pp
TicketStability@1.0pp
TicketStability@2.0pp
DecisionStability por jogo
DoubleSetStability
Top1SetStability
MarkStability
```

Objetivo:

```text
distinguir decisão ótima de decisão robustamente ótima
```

---

# Auditoria histórica do P13+ Regret — PRIORIDADE 2

Aplicar o regret retrospectivamente aos concursos históricos.

Para cada decisão, registrar:

```text
regret absoluto
regret relativo
classe de robustez
decisão escolhida
resultado real
```

Perguntas a responder:

```text
decisões de baixo regret erram mais?
decisões de alto regret são mais estáveis?
duplos de baixo regret trocam mais sob perturbação?
secos de alto regret têm maior taxa real de acerto?
```

O objetivo é verificar se o regret possui valor interpretativo real, além de valor matemático.

---

# Auditoria histórica D12 / D13 / D23 — PRIORIDADE 3

Como D23 elimina completamente o Top1, deve ser auditado separadamente.

Por tipo:

```text
n
cobertura prevista
cobertura observada
acertos
lift
```

Para D23:

```text
Top1 acertou
Top2 acertou
Top3 acertou
D23 evitou erro do seco
D23 causou erro ao excluir Top1
RecoveryWins
RecoveryLosses
NetRecoveryGain
RecoveryRate
```

Pergunta central:

```text
abandonar o favorito nominal via D23 gera ganho consistente fora da amostra?
```

---

# Auditoria histórica da fronteira 6º × 7º

A fronteira simples por `1-p(top1)` é somente diagnóstico.

Avaliar concursos históricos com:

```text
margem <= 0.01
margem <= 0.02
margem <= 0.03
margem <= 0.05
```

Usar `P13plusRegret` para medir a verdadeira distância estrutural entre decisão escolhida e melhor alternativa global.

Nenhum `cutoff_score` deve ser implantado antes de validação Champion/Challenger fora da amostra.

---

# Simulação conjunta de incerteza — PRIORIDADE FUTURA

Depois das perturbações determinísticas, gerar múltiplos cenários probabilísticos plausíveis:

```text
probabilidades originais
+ ruído pequeno controlado
-> renormalização
-> reotimização
```

Exemplo inicial:

```text
500 a 1.000 cenários
```

Relatório:

```text
Jogo
Decisão mais frequente
Frequência da decisão atual
Frequência de seco
Frequência de D12
Frequência de D13
Frequência de D23
```

Isso produz um **consenso do otimizador sob incerteza**.

Monte Carlo deve ser usado apenas para estabilidade/diagnóstico, nunca para substituir o cálculo exato de `P(>=13)`.

---

# TicketFragility — diagnóstico futuro

Criar uma medida agregada de quão perto o bilhete está de alternativas estruturais competitivas.

Sugestão inicial:

```text
Bottom4RegretMean
= média dos quatro menores regrets relativos do bilhete
```

Interpretação:

```text
baixo -> ótimo depende de várias decisões de fronteira
alto  -> ótimo está bem separado das alternativas
```

Também considerar:

```text
MinRegret
MedianRegret
Bottom6RegretMean
```

`TicketFragility` deve permanecer como telemetria até existir validação histórica suficiente.

---

# Índice de fragilidade por decisão — diagnóstico futuro

Depois de implementar estabilidade por perturbações, combinar:

```text
P13plusRegret
DecisionStability
probability gaps
```

Saída possível:

```text
MUITO ROBUSTA
ROBUSTA
MODERADA
FRÁGIL
FRONTEIRA
```

O índice não deve alterar o bilhete sem validação Champion/Challenger.

---

# Relatório automático “por que esta decisão?”

A telemetria deve evoluir para explicações quantitativas.

Exemplo:

```text
Jogo 8 recebeu D23 porque:
- pTop1 baixo
- gap12 pequeno
- RecoveryGain elevado
- alto P13plusRegret se D23 for proibido
- alta DecisionStability sob perturbações

Classificação: MUITO ROBUSTA
```

Para um seco:

```text
Jogo A permaneceu seco porque promover esse jogo a duplo
forçaria uma reorganização estrutural cuja melhor solução
reduziria P(>=13) em X.
```

Sempre que possível, usar regret e estabilidade em vez de justificativas exclusivamente heurísticas.

---

# Backtest walk-forward multi-janelas — PRIORIDADE ALTA

Além do holdout cronológico 80/20, implementar validação expanding-window.

```text
Treino 1..200 -> Teste 201..220
Treino 1..220 -> Teste 221..240
Treino 1..240 -> Teste 241..260
...
```

Comparar:

```text
A = probabilidades brutas
B = + temperatura
C = + temperatura + risk_rank
D = Champion atual
E = Challenger em avaliação
```

Relatório mínimo:

```text
Concursos
LogLoss
Brier
ECE
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

A promoção deve privilegiar consistência entre múltiplas janelas, e não desempenho excepcional em um único holdout.

---

# Bootstrap pareado e incerteza

Como `>=13` é raro, diferenças pequenas podem ser ruído.

Para Champion/Challenger:

```text
IC95% do delta de mean_hits
IC95% do DecisionNetGain
IC95% do Net13Gain
P(DecisionNetGain > 0)
P(Net13Gain > 0)
```

Preservar o pareamento por concurso.

Inicialmente essas métricas são diagnóstico, não Hard Gate.

---

# Champion / Challenger formal

Toda nova regra, modelo ou heurística deve entrar primeiro como Challenger.

```text
Champion = versão atualmente implantada
Challenger = técnica candidata
```

Relatório padrão:

```text
Métrica                  Champion   Challenger   Delta
LogLoss
Brier
ECE
mean_hits
12+
13+
14
Net13Gain
DecisionNetGain
DecisionWinRate
DecisionLossRate
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

# Backtest de estruturas alternativas

A estrutura 10-5-5 é Hard Constraint do Champion atual, mas pode ser estudada em experimentos separados.

Exemplos:

```text
10-5-5
9-6-5
9-5-6
11-5-4
11-4-5
```

Comparar fora da amostra:

```text
13+
12+
mean_hits
Net13Gain
DecisionNetGain
```

A comparação serve para testar a justificativa empírica da estrutura, não para relaxá-la silenciosamente.

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

Não promover uma composição fixa apenas porque aparece frequentemente no otimizador.

---

# StructuralCost

Somente auditoria:

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

Treinar um Challenger somente nos jogos históricos com `top1_hit = 0`.

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

Possíveis evoluções:

```text
f(risk_rank)
```

ou:

```text
f(pTop1, risk_percentile)
```

Isso pode reduzir descontinuidades artificiais entre ranks vizinhos.

Só testar depois que walk-forward, bootstrap e Champion/Challenger estiverem consolidados.

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
P13plusRegret absoluto
P13plusRegret relativo
melhor alternativa global
DecisionStability, quando disponível
FragilityClass, quando disponível
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
TicketFragility, quando disponível
```

---

# Roadmap priorizado

## Concluído recentemente

```text
P13+ Regret por jogo/decisão
reotimização global contrafactual
regret absoluto e relativo
classificação inicial de robustez
```

## Prioridade 1 — robustez probabilística

```text
perturbações ±0,5 / ±1 / ±2 pp
DecisionStability
DoubleSetStability
TicketStability
MarkStability
```

## Prioridade 2 — aprofundar o P13+ Regret

```text
RegretSeco / RegretD12 / RegretD13 / RegretD23
classificação granular de robustez
auditoria histórica do regret
motor genérico optimize_with_constraint
```

## Prioridade 3 — validação dos tipos de duplo

```text
auditoria histórica D12/D13/D23
RecoveryWins / RecoveryLosses
NetRecoveryGain
RecoveryRate
```

## Prioridade 4 — validação do risk_rank

```text
RiskRankPrecision@6
RiskRankRecall@6
RiskRankNDCG@6
RiskRankECE
RiskRankBrier
```

## Prioridade 5 — validação temporal

```text
walk-forward multi-janelas
bootstrap pareado
Champion/Challenger padronizado
```

## Prioridade 6 — explicabilidade

```text
relatório automático “por que esta decisão?”
TicketFragility
índice de fragilidade por decisão
```

## Prioridade 7 — pesquisa estrutural

```text
auditoria histórica 6º x 7º
backtest de estruturas alternativas
StructuralCost
Top1_fail -> Top2/Top3
```

## Prioridade futura

```text
simulação conjunta de incerteza
risk_rank suavizado
modelos mais complexos somente após evidência clara de necessidade
```

---

# O que evitar por enquanto

Evitar adicionar complexidade sem evidência fora da amostra:

```text
scores manuais para escolher duplos
bônus arbitrário a zebras
pesos arbitrários de entropia
cutoff_score não validado
redes neurais apenas por sofisticação
XGBoost/LightGBM sem baseline convincente
calibrações excessivamente segmentadas
14x3 ou mais parâmetros sem shrinkage
novas Soft Constraints sem backtest
Monte Carlo onde a solução exata já existe
```

Antes de criar nova heurística para os seis duplos, o projeto deve primeiro **medir regret, estabilidade e robustez das decisões do otimizador atual**.

---

# Princípio geral

O projeto procura construir **um único bilhete de 14 jogos**, com exatamente **8 secos, 6 duplos, 0 triplos e distribuição 10-5-5**, cuja combinação de probabilidades, histórico e estrutura maximize:

```text
P(acertos >= 13)
```

A evolução deve seguir:

```text
medir
-> auditar
-> explicar
-> criar Challenger
-> validar fora da amostra
-> medir incerteza
-> promover somente se houver evidência
```

**Toda melhoria deve ser demonstrada fora da amostra e sempre dentro das Hard Constraints.**