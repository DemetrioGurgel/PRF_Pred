# Predição semanal de acidentes – PRF

Projeto em Python/notebook para prever **se, na semana seguinte**, um determinado **segmento de rodovia (UF, BR, km em blocos de 5 km)** vai ter **ao menos 1 acidente**. A saída prática é uma **lista semanal de prioridades (Top-N)** para fiscalização/educação.

> ✅ O notebook já faz: limpeza → agregação semanal densa → engenharia de atributos → treino com split temporal → comparação de modelos → gráficos → exports (CSV).

---

## 1. Objetivo

- Transformar histórico de ocorrências da PRF em **sinal preditivo**.
- **Priorizar segmentos** de rodovia por semana (Top-30 geral e Top-5 por UF).
- Mostrar o **trade-off** entre “acertar muito” (cobertura 1%) e “pegar mais casos” (cobertura 5%).

---

## 2. Fonte e formato dos dados

O notebook espera uma base já limpa, algo como:

- `prf_clean.csv.gz` (ou equivalente)
- colunas mínimas:
  - `data` (datetime)
  - `uf`
  - `br`
  - `km`
- colunas úteis/opcionais:
  - `hora` **ou** `segundos_desde_00` (pra marcar noite)
  - `latitude`, `longitude`
  - atributos do veículo / pessoa (idade, ano_fabricacao_veiculo, tipo_veiculo, tipo_envolvido, etc.)

O próprio notebook tem blocos de **normalização/compactação** (downcast, remover texto redundante, salvar parquet/csv.gz robusto).

---

## 3. Pipeline feito no notebook

1. **Pré-processamento**
   - remove duplicatas óbvias
   - corrige faixas inválidas (`km`, `br`, `idade`, `lat/lon`)
   - normaliza strings/categorias
   - cria `km_bin5 = floor(km / 5) * 5` (segmento de 5 km)
   - cria flags de missing

2. **Agregação semanal densa**
   - grupo por: `uf`, `br`, `km_bin5`, `sem_ini` (segunda-feira)
   - **reindexa a série** para não perder semanas sem acidente (entra como `acidentes=0`)
   - calcula `% de ocorrências noturnas` na semana

3. **Alvo causal (t+1)**
   - para cada segmento:  
     `y_next = 1` se **na semana seguinte** houve ≥1 acidente ali (isso evita vazamento de futuro).

4. **Engenharia de atributos**
   - lags: 1, 4, 12 semanas
   - rollings: 4, 12, 24 e **52** semanas
   - EWMA causal
   - recência: `since_last_acc`, `cnt_last4`, `cnt_last12`
   - sazonalidade: seno/cosseno de mês e semana do ano
   - **vizinhança**: junta info dos km vizinhos (±5 e ±10 km) no mesmo (UF, BR, semana)
   - **feriados nacionais** (na semana e na próxima)

5. **Split temporal**
   - treino: **tudo antes de 2024**
   - teste (simulando produção): **2024**

6. **Modelos treinados**
   - `HistGradientBoostingClassifier` (modelo principal)
   - Regressão Logística **elastic net** + **calibração isotônica**
   - KNN **geo-temporal** (features enxutas)
   - Ensemble simples: **0.7 * Boosting + 0.3 * LogReg(cal)**

7. **Métricas e gráficos**
   - ROC e Precision-Recall comparando modelos
   - Curva de **calibração** (mostrando que a logística calibrada fica colada na reta 45°)
   - **Cumulative Gain**
   - **P@K / Lift@K** (50, 100, 200, 500, 1000)
   - **Precisão semanal do Top-30** (linha do tempo 2024)
   - **Heatmap** do Top-N de uma semana → “onde ir”
   - **Matrizes de confusão**:
     - no limiar ótimo de F1
     - cobertura ≈ **1%**
     - cobertura ≈ **5%**

8. **Exports**
   - `scores_teste_completos.csv.gz`
   - `top30_por_semana.csv`
   - `quota_top5_por_uf_semana.csv`
   - `feature_importance_test_perm.csv`
   - + versões por modelo (`top30_boosting.csv`, `top30_logreg_cal.csv`, etc.)

---


## 4. Resultados (resumo)

- **Boosting**: ROC-AUC ≈ **0.74–0.75**, PR-AUC ≈ **0.45–0.46**
- **LogReg (cal)**: ROC-AUC ≈ **0.74**, PR-AUC ≈ **0.44**
- **Ensemble**: colado no Boosting
- **Top-30 por semana (2024)**: quase sempre precisão **≥ 0.85**, várias semanas batendo **~1.0**
- Em cobertura **1%**, precisão **muito alta** (bom pra operação), mas recall baixo → trade-off visível nas matrizes

---
