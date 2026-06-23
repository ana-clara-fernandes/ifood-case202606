# ifood-case202606
# iFood Data Science Case — Uplift Modeling for Offer Targeting

> Case técnico desenvolvido como parte do processo seletivo para a posição de Data Scientist no iFood.

---

## 🎯 Problema

O objetivo do case é identificar **quais clientes devem receber cada tipo de oferta** (desconto, BOGO ou informacional) de forma a maximizar a taxa de conversão e minimizar o custo de campanhas — evitando desperdiçar incentivos em clientes que converteriam de qualquer forma (*Sure Things*).

A abordagem escolhida é **Uplift Modeling via S-Learner**, que estima o efeito causal incremental de cada tipo de oferta sobre a probabilidade de conversão de cada cliente.

---

## 🧪 Metodologia

### Arquitetura: S-Learner com LightGBM

Optou-se pelo **S-Learner** — um único modelo que recebe o tipo de oferta como feature regular — em detrimento dos meta-learners mais complexos (T-Learner, X-Learner), por três razões:

1. **Benchmark empírico**: UpliftBench (2025) demonstrou que o S-Learner com LightGBM superou o X-Learner na maior parte dos datasets avaliados (Qini Score: 0.3759 vs. 0.3614 no Criteo-13M).
2. **Dado quasi-experimental**: os grupos experimentais foram validados como balanceados (Kruskal-Wallis p > 0.47 para todas as features), tornando o ajuste de confundimento do X-Learner desnecessário.
3. **Parsimônia**: a diferença de conversão entre grupos já era estatisticamente significativa (χ² = 96.48, p < 0.0001), confirmando sinal forte no dado observacional.

### Fundamento teórico

- **Lo (2002)** — "The True Lift Model" (ACM SIGKDD Explorations): referência fundacional para uplift via regressão única com tratamento como feature.
- **Shchetkina & Berman (2024)** — "When Is Heterogeneity Actionable for Targeting?" (EC '24, ACM): guia para decidir quando a heterogeneidade do efeito de tratamento justifica targeting individual.

---

## 📊 Resultados

| Métrica | Valor |
|---|---|
| AUC-ROC | **0.793** |
| Calibração | Excelente (curva de calibração aderente) |
| Distribuição de recomendação | Desconto: 78.6% · Informacional: 21.4% · BOGO: 0% |
| Ganho de conversão (vs. baseline) | **+5.1 p.p.** |
| Economia estimada por campanha | **R$ 20.600** (excluindo Sure Things — 28.7% da base) |
| ROI imediato | 0.63x (positivo no horizonte de LTV, 2–3 recompras) |

**Achado principal:** o tipo de oferta ranked **9º de 12 features** por importância (gain) — o perfil do cliente (tenure, canal de cadastro, limite de crédito, idade) prediz conversão de forma muito mais determinante do que o incentivo oferecido. Isso implica que targeting inteligente de *quem* recebe oferta tem mais valor do que otimizar *qual* oferta.

> O BOGO nunca foi recomendado — em todos os segmentos, o desconto domina o BOGO na estimativa de uplift.

---

## 🛠️ Stack Tecnológica

| Etapa | Ferramenta |
|---|---|
| Processamento de dados | **PySpark** no **Databricks** (Free Edition / Serverless) |
| Armazenamento | **Unity Catalog** (`workspace.default.ifood_case`) |
| Modelagem | **LightGBM** + scikit-learn (Jupyter local) |
| Análise estatística | SciPy (Kruskal-Wallis, chi-quadrado), Cramér's V |
| Visualização | Matplotlib, Seaborn |

---

## 📁 Estrutura do Repositório

```
ifood-data-science-case/
│
├── data/
│   └── raw/                  # Dados brutos (offers.json, profile.json, transactions.json)
│
├── notebooks/
│   ├── 01_processing.ipynb   # Limpeza e engenharia de features (PySpark / Databricks)
│   └── 02_modeling.ipynb     # EDA, modelagem uplift e simulação contrafactual
│
├── presentation/             # Apresentação executiva (5 slides)
│
├── .gitignore
├── requirements.txt
└── README.md
```

---

## ⚙️ Como Rodar

### Pré-requisitos

- Python 3.10+
- Databricks Free Edition (para o notebook de processamento)
- Jupyter Notebook ou JupyterLab (para o notebook de modelagem)

### Instalação

```bash
git clone https://github.com/SEU_USUARIO/ifood-data-science-case.git
cd ifood-data-science-case
pip install -r requirements.txt
```

### Notebook 1 — Processamento (PySpark / Databricks)

1. Faça upload dos dados brutos (`offers.json`, `profile.json`, `transactions.json`) no Databricks Unity Catalog Volume.
2. Importe `notebooks/01_processing.ipynb` no seu workspace Databricks via **Git Folders**.
3. Execute todas as células em ordem — o output é a Delta Table `workspace.default.ifood_final_dataset` (76.277 linhas, 27 colunas).

### Notebook 2 — Modelagem (Jupyter local)

```bash
jupyter notebook notebooks/02_modeling.ipynb
```

> O notebook de modelagem lê os dados processados exportados como CSV/Parquet. Certifique-se de exportar `ifood_final_dataset` do Databricks antes de rodar.

---

## 🔍 Decisões Técnicas Relevantes

- **Sentinel de idade (valor 118):** correlacionado perfeitamente com `gender = null` e `credit_card_limit = null`. Substituído por `null` com criação de flag binária `age_missing`.
- **Duplicatas em `offer_completed`:** 397 linhas duplicadas concentradas exclusivamente nesse evento — possível double-bonus concedido por erro. Preservadas em tabela de auditoria `transactions_duplicates_removed`.
- **Coluna `value` heterogênea:** dois schemas distintos para `offer_id` (`offer id` com espaço nos eventos received/viewed vs. `offer_id` com underscore no completed), unificados via `coalesce`.
- **Ofertas informacionais:** sem evento `offer_completed`, o sucesso foi redefinido como *viewed + transaction dentro da janela de validade*.
- **ROI conservador:** calculado usando comparação observed-vs-observed (não contrafactual), resultando em 0.63x imediato — com documentação explícita de que o break-even ocorre no horizonte de LTV.

---

## 📚 Referências

- Lo, V. S. Y. (2002). The True Lift Model — A Novel Data Mining Approach to Response Modeling in Database Marketing. *ACM SIGKDD Explorations*, 4(2), 78–86.
- Shchetkina, A., & Berman, R. (2024). When Is Heterogeneity Actionable for Targeting? *EC '24: Proceedings of the 25th ACM Conference on Economics and Computation*.
- Zhao, Z. et al. (2025). UpliftBench: A Large-Scale Empirical Comparison of Meta-Learners and Causal Forests. *arXiv preprint*.
